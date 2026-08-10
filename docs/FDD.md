# FDD — Sistema de Webhooks de Notificação de Pedidos

## Contexto e motivação técnica

Este documento detalha o "como construir" da feature de Webhooks proposta em
[RFC](./RFC.md) e decidida nos [ADRs](./adrs/README.md). É o nível de detalhamento
suficiente para um desenvolvedor começar a implementar sem precisar reabrir a
transcrição da reunião. Toda referência a arquivo de código é um caminho real,
confirmado no repositório-base (ver `../TRACKER.md` para a rastreabilidade completa).

## Objetivos técnicos

- Emitir um evento de webhook por endpoint elegível (inscrito no `toStatus`) a cada
  mudança de status de pedido relevante — um customer com múltiplos endpoints
  inscritos no mesmo status recebe um evento por endpoint, não um único evento
  compartilhado — de forma atômica com a transação que efetiva essa mudança.
- Entregar o evento ao endpoint do cliente em, no pior caso determinístico, poucos
  segundos (ciclo de polling do worker), respeitando o teto de <10s combinado com os
  clientes (`TRANSCRICAO.md` [09:02] Marcos).
- Garantir at-least-once delivery com meios de deduplicação do lado do cliente.
- Não introduzir nenhuma peça de infraestrutura nova além do MySQL e do runtime
  Node.js já usados pelo OMS.
- Não degradar a robustez nem a performance da transação `changeStatus` já existente.

## Escopo e exclusões

**Dentro do escopo:** CRUD de configuração de webhook; emissão via outbox; worker de
entrega com retry/backoff/DLQ; assinatura HMAC; histórico de entregas; replay manual
de DLQ.

**Fora do escopo** (lista completa e origem de cada item em
[PRD — Fora de escopo](./PRD.md#fora-de-escopo)): notificação proativa por e-mail ao
cliente; rate limiting de saída; dashboard visual; múltiplos workers em paralelo;
arquivamento de eventos entregues. Rate limiting de saída, e-mail e múltiplos workers
também aparecem em [RFC — Questões em aberto](./RFC.md#questões-em-aberto), por ainda
não terem uma direção decidida; dashboard visual e arquivamento foram descartados de
forma direta na reunião (não são questões em aberto, não constam no RFC).

## Fluxos detalhados

### 1. Criação do evento na Outbox

1. `OrderController.changeStatus` chama `OrderService.changeStatus` (já existente em
   `src/modules/orders/order.service.ts`), sem alteração de assinatura pública.
2. Dentro do `this.prisma.$transaction` já existente nesse método — após validar a
   transição (`canTransition`, `src/modules/orders/order.status.ts`) e antes do commit
   — o service chama `publishWebhookEvent(tx, order, fromStatus, toStatus)`, exportada
   pelo novo módulo `src/modules/webhooks/webhook.outbox.ts`.
3. `publishWebhookEvent` busca (via `tx`) os `webhook_endpoint` ativos do
   `customer_id` do pedido cujo `events` inclui `toStatus`. Se nenhum endpoint estiver
   inscrito naquele status, **nada é inserido** — a filtragem ocorre na inserção, não
   no envio (`TRANSCRICAO.md` [09:33][09:34] Bruno, Diego).
4. Para cada endpoint elegível, insere uma linha em `webhook_outbox` com:
   `id` (UUID), `webhook_endpoint_id`, `event_id` (UUID, vira `X-Event-Id`),
   `event_type = "order.status_changed"`, `payload` (JSON já renderizado — snapshot do
   estado no momento da inserção, `TRANSCRICAO.md` [09:52] Larissa/Diego/Bruno),
   `status = "pending"`, `attempts = 0`, `created_at`.
5. Se a inserção falhar, a exceção propaga e todo o `$transaction` de `changeStatus`
   sofre rollback — status do pedido não muda sem o evento correspondente
   (`TRANSCRICAO.md` [09:40][09:41] Bruno, Diego).

### 2. Processamento pelo Worker

1. `src/worker.ts` (novo entry-point, análogo a `src/server.ts`) inicializa seu
   próprio `PrismaClient` (mesma `DATABASE_URL`, processo separado —
   `TRANSCRICAO.md` [09:29][09:30] Diego, Bruno) e inicia um loop de polling a cada
   2000ms.
2. A cada ciclo, `webhook.worker.ts` busca em `webhook_outbox` os eventos com
   `status = "pending"` e `next_attempt_at <= now()` (na primeira tentativa,
   `next_attempt_at` é nulo/já vencido), ordenados por `created_at`, em lote pequeno
   (ex.: 20). O evento permanece com `status = "pending"` durante toda a janela de
   retry — não existe um status intermediário `"failed"` neste modelo; ver Seção 3.
3. Para cada evento: monta os headers (ver "Contratos públicos"), calcula HMAC sobre
   o `payload`, faz `POST` para a `url` do `webhook_endpoint`, com timeout de 10s
   (`TRANSCRICAO.md` [09:42] Diego).
4. Resposta 2xx dentro do timeout → marca `status = "delivered"`, grava a entrega em
   `webhook_delivery` (para o endpoint de histórico).
5. Timeout ou resposta não-2xx → segue para o fluxo de retry.

### 3. Retry e backoff

1. Em falha, incrementa `attempts` e calcula `next_attempt_at = now() + backoff[attempts]`,
   onde `backoff = [1min, 5min, 30min, 2h, 12h]` (`TRANSCRICAO.md` [09:17] Diego).
2. Evento permanece com `status = "pending"` até `attempts` esgotar as 5 tentativas.
3. Cada tentativa (sucesso ou falha) é registrada em `webhook_delivery` (payload
   enviado, status HTTP recebido/timeout, tempo de resposta), alimentando
   `GET /webhooks/:id/deliveries`.

### 4. Dead Letter Queue (DLQ)

1. Após a 5ª tentativa falhar, o worker **remove a linha de `webhook_outbox` e insere
   uma nova em `webhook_dead_letter`** (tabela separada, decisão fechada em
   [ADR-003](./adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) — não um status
   dentro da própria outbox, ver alternativa descartada no ADR), mantendo `payload`,
   `reason` da última falha e `failed_at` (`TRANSCRICAO.md` [09:18] Diego).
2. `POST /admin/webhooks/dead-letter/:id/replay` (role `ADMIN`,
   `requireRole('ADMIN')` de `src/middlewares/auth.middleware.ts`) recoloca o evento em
   `webhook_outbox` com `status = "pending"` e `attempts = 0`. A ação é logada via
   `logger` (Pino) com o `userId` de quem executou o replay, para auditoria
   (`TRANSCRICAO.md` [09:35][09:36] Larissa, Sofia).

## Contratos públicos

Todos os endpoints ficam sob `/api/v1/webhooks`, agregados em
`src/routes/index.ts` (`router.use('/webhooks', buildWebhookRouter(...))`,
seguindo o padrão de `buildOrderRouter`). CRUD exige apenas `authenticate` (JWT
normal); `customer_id` vem do body/path, não do JWT, pois o JWT é de usuário
operador, não do cliente (`TRANSCRICAO.md` [09:32] Marcos, Larissa).

### `POST /api/v1/webhooks`

Cria um endpoint de webhook. A secret é gerada pelo servidor e devolvida **apenas**
nesta resposta (`TRANSCRICAO.md` [09:31] Marcos).

Request:
```json
{
  "customerId": "b3f1b2e0-1111-4a2b-9c3d-000000000001",
  "url": "https://cliente.example.com/webhooks/oms",
  "events": ["SHIPPED", "DELIVERED"]
}
```

Response `201 Created`:
```json
{
  "id": "2b0e6f2a-2222-4a2b-9c3d-000000000002",
  "customerId": "b3f1b2e0-1111-4a2b-9c3d-000000000001",
  "url": "https://cliente.example.com/webhooks/oms",
  "events": ["SHIPPED", "DELIVERED"],
  "secret": "whsec_5f8a1c...",
  "active": true,
  "createdAt": "2026-08-09T12:00:00.000Z"
}
```

Erros possíveis: `400 WEBHOOK_INVALID_URL` (URL não é `https`), `400
WEBHOOK_INVALID_EVENTS` (status inexistente na lista), `404 NOT_FOUND` (`customerId`
não existe).

### `GET /api/v1/webhooks?customerId=...`

Lista os webhooks de um customer, paginado (reuso de `paginated` /
`src/shared/http/response.ts`).

Response `200 OK`:
```json
{
  "data": [
    {
      "id": "2b0e6f2a-2222-4a2b-9c3d-000000000002",
      "customerId": "b3f1b2e0-1111-4a2b-9c3d-000000000001",
      "url": "https://cliente.example.com/webhooks/oms",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

Observação: a `secret` **não** é retornada em listagem/consulta, apenas na criação e
na rotação — decisão de implementação necessária para não expor segredo em endpoint de
leitura; não foi discutida explicitamente na reunião (classificar como inferência de
segurança, não decisão registrada em ata).

### `PATCH /api/v1/webhooks/:id`

Edita `url` e/ou `events`.

Request:
```json
{ "events": ["PAID", "SHIPPED", "DELIVERED"] }
```

Response `200 OK`:
```json
{
  "id": "2b0e6f2a-2222-4a2b-9c3d-000000000002",
  "customerId": "b3f1b2e0-1111-4a2b-9c3d-000000000001",
  "url": "https://cliente.example.com/webhooks/oms",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true
}
```
Mesmo formato de objeto webhook usado em `GET /api/v1/webhooks` (sem o campo
`secret`, pelos mesmos motivos descritos acima). Erros: `404 WEBHOOK_NOT_FOUND`,
`400 WEBHOOK_INVALID_URL`.

### `DELETE /api/v1/webhooks/:id`

Remove o cadastro. Response `204 No Content`. Erro: `404 WEBHOOK_NOT_FOUND`.

### `GET /api/v1/webhooks/:id/deliveries`

Histórico das últimas 100 entregas (`TRANSCRICAO.md` [09:34] Marcos).

Response `200 OK`:
```json
{
  "data": [
    {
      "id": "9c11...",
      "eventId": "b7a0...",
      "eventType": "order.status_changed",
      "attempt": 1,
      "success": true,
      "requestPayload": { "event_id": "b7a0...", "event_type": "order.status_changed", "...": "..." },
      "responseStatus": 200,
      "responseBody": "OK",
      "responseTimeMs": 184,
      "deliveredAt": "2026-08-09T12:00:05.120Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 1, "totalPages": 1 }
}
```

`requestPayload` e `responseBody` são obrigatórios no exemplo porque o requisito
original pede explicitamente "payload, response" no histórico, não só o status
(`TRANSCRICAO.md` [09:34] Marcos) — ver `PRD-FR-09`. `responseBody` é truncado/limitado
em tamanho na persistência (detalhe de implementação, não decidido em ata).

### `POST /api/v1/admin/webhooks/dead-letter/:id/replay`

Exige role `ADMIN`. Recoloca o evento na outbox como pendente
(`TRANSCRICAO.md` [09:18][09:35] Diego, Larissa).

Response `202 Accepted`:
```json
{ "outboxId": "af21...", "status": "pending", "attempts": 0 }
```

Erro: `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`, `403 FORBIDDEN` (role insuficiente, via
`ForbiddenError` já existente em `src/shared/errors/http-errors.ts`).

### Payload de entrega ao cliente

Enviado pelo worker ao `url` do endpoint cadastrado:

```json
{
  "event_id": "b7a0c2e4-3333-4a2b-9c3d-000000000003",
  "event_type": "order.status_changed",
  "timestamp": "2026-08-09T12:00:00.000Z",
  "order_id": "1a2b3c4d-4444-4a2b-9c3d-000000000004",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "b3f1b2e0-1111-4a2b-9c3d-000000000001",
  "total_cents": 15990
}
```

Campos e exclusão de `items` conforme `TRANSCRICAO.md` [09:43] Diego (payload enxuto;
cliente busca detalhe via `GET /orders/:id` se precisar).

Headers (`TRANSCRICAO.md` [09:44] Diego, Sofia):

| Header | Conteúdo |
| --- | --- |
| `Content-Type` | `application/json` |
| `X-Event-Id` | UUID do evento (dedup do lado do cliente) |
| `X-Signature` | HMAC-SHA256 do corpo, hex, usando a secret do endpoint |
| `X-Timestamp` | timestamp ISO 8601 do envio (detecção de replay attack) |
| `X-Webhook-Id` | ID do `webhook_endpoint` cadastrado |

## Matriz de erros (`WEBHOOK_*`)

Apenas 3 destes códigos foram citados literalmente na reunião — `WEBHOOK_NOT_FOUND`,
`WEBHOOK_INVALID_URL` e `WEBHOOK_SECRET_REQUIRED` — como exemplos do padrão de
convenção de nomenclatura ("Códigos tipo [...], etc.", `TRANSCRICAO.md` [09:28]
Bruno). Os demais (`WEBHOOK_INVALID_EVENTS`, `WEBHOOK_PAYLOAD_TOO_LARGE`,
`WEBHOOK_DEAD_LETTER_NOT_FOUND`, `WEBHOOK_DELIVERY_TIMEOUT`) são **inferência de
implementação**: nomes que seguem a mesma convenção de prefixo decidida em ata, mas
que ninguém pronunciou — não confundir com decisão registrada.

| Código | Status HTTP | Quando ocorre | Origem do nome |
| --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | `id` de webhook inexistente em GET/PATCH/DELETE | Literal (`TRANSCRICAO.md` [09:28] Bruno) |
| `WEBHOOK_INVALID_URL` | 400 | URL cadastrada não é `https` (`TRANSCRICAO.md` [09:23] Sofia) | Literal (`TRANSCRICAO.md` [09:28] Bruno) |
| `WEBHOOK_SECRET_REQUIRED` | 400 | Operação que depende de secret sem secret configurada | Literal (`TRANSCRICAO.md` [09:28] Bruno); cenário de uso inferido |
| `WEBHOOK_INVALID_EVENTS` | 400 | Lista `events` contém valor fora do enum `OrderStatus` | Inferência — segue o prefixo, nome não dito em ata |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | — (uso interno do worker, não HTTP síncrono) | Payload do evento excede 64KB no momento do envio (`TRANSCRICAO.md` [09:23][09:24] Sofia, Diego) | Inferência — comportamento (erro acima de 64KB) é decisão de ata; nome do código não |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `id` inexistente em `POST /admin/webhooks/dead-letter/:id/replay` | Inferência — segue o prefixo, nome não dito em ata |
| `WEBHOOK_DELIVERY_TIMEOUT` | — (uso interno do worker, não HTTP síncrono) | Chamada ao endpoint do cliente excede 10s (`TRANSCRICAO.md` [09:42] Diego) | Inferência — comportamento (timeout de 10s) é decisão de ata; nome do código não |

Erros de validação de schema (Zod) e de autenticação/autorização reaproveitam os
códigos já existentes (`VALIDATION_ERROR`, `UNAUTHORIZED`, `FORBIDDEN`, `NOT_FOUND`),
tratados sem alteração pelo `errorMiddleware` (`src/middlewares/error.middleware.ts`).

## Estratégias de resiliência

- **Timeout:** 10s por chamada HTTP do worker (`TRANSCRICAO.md` [09:42] Diego).
- **Retry:** 5 tentativas, backoff 1m/5m/30m/2h/12h (ver
  [ADR-003](./adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)).
- **Fallback:** após esgotar as tentativas, evento vai para DLQ; reprocessamento é
  manual, não automático (`TRANSCRICAO.md` [09:18] Diego). Não há fallback de canal
  (ex.: e-mail) nesta fase — explicitamente fora de escopo.
- **Isolamento de falha:** falha de entrega a um cliente não afeta a entrega a outros
  clientes nem a disponibilidade da API principal, pois o worker roda em processo e
  loop próprios.

## Observabilidade

- **Logs:** reuso da instância Pino existente (`src/shared/logger/index.ts`). Worker
  loga por ciclo de polling (quantidade de eventos processados) e por tentativa de
  entrega (sucesso/falha, `event_id`, `webhook_endpoint_id`, `responseStatus` ou erro),
  seguindo o mesmo padrão de campos estruturados usado em
  `src/middlewares/request-logger.middleware.ts` (`requestId`-equivalente por evento).
- **Métricas** (a expor via contagem/instrumentação leve, reaproveitando o mesmo
  processo de logging já que o projeto não tem um coletor de métricas dedicado hoje):
  eventos inseridos na outbox por minuto; taxa de entregas com sucesso na 1ª tentativa;
  eventos em `dead_letter` por hora; tempo médio de resposta dos endpoints de cliente
  (já coletado por entrega, via `webhook_delivery.responseTimeMs`).
- **Tracing:** reuso do padrão de correlação já existente
  (`X-Request-Id`/`req.id` em `request-logger.middleware.ts`) para requisições HTTP de
  entrada do CRUD de webhooks; para o fluxo assíncrono do worker, o `event_id`
  (`X-Event-Id`) funciona como identificador de correlação ponta a ponta (da inserção
  na outbox até a entrega/DLQ), permitindo reconstruir o trajeto de um evento nos logs.

## Dependências e compatibilidade

- Depende do MySQL 8 já provisionado (`docker-compose.yml`) e do `PrismaClient`
  (`@prisma/client` 5.22.0, já em `package.json`).
- Novo script `npm run worker` (análogo a `npm run dev`/`start`), executando
  `src/worker.ts`.
- Não requer nenhuma nova dependência de terceiros para HMAC (`crypto` nativo do
  Node.js cobre HMAC-SHA256).
- Não altera nenhum endpoint existente de `orders`, `customers`, `products`, `users`
  ou `auth` — é aditivo.

## Integração com o sistema existente

Seção obrigatória — pelo menos 4 caminhos reais do código-base e como a feature se
integra com cada um:

1. **`src/modules/orders/order.service.ts`** (método `changeStatus`): ponto de
   integração central. A transação existente passa a chamar
   `publishWebhookEvent(tx, order, fromStatus, toStatus)` antes do commit, inserindo o
   evento na outbox como parte atômica da mesma mudança de status. Nenhuma alteração
   na assinatura pública do método nem no contrato de resposta ao cliente da API de
   pedidos.
2. **`src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts`**: todas as
   novas classes de erro do módulo webhooks (ex.: `WebhookNotFoundError`,
   `InvalidWebhookUrlError`) estendem `AppError`/as classes HTTP existentes
   (`NotFoundError`, `BadRequestError`, `UnprocessableEntityError`), seguindo
   exatamente o padrão de `InsufficientStockError`/`InvalidStatusTransitionError` já
   usado no módulo de orders.
3. **`src/middlewares/error.middleware.ts`**: nenhuma alteração de código necessária —
   como todos os novos erros são instâncias de `AppError`, já são capturados pelo
   branch existente (`if (err instanceof AppError)`), retornando o mesmo formato
   `{ error: { code, message, details } }`.
4. **`src/middlewares/auth.middleware.ts`**: `authenticate` protege todo o CRUD de
   webhook; `requireRole('ADMIN')` protege especificamente o endpoint de replay de
   DLQ, reaproveitando a mesma função usada hoje para restringir operações
   sensíveis, sem qualquer alteração no middleware.
5. **`src/routes/index.ts`**: novo `router.use('/webhooks', buildWebhookRouter(controllers.webhooks))`
   e `router.use('/admin/webhooks', buildWebhookAdminRouter(controllers.webhooks))`
   adicionados ao mesmo agregador de rotas que já registra `orders`, `customers`,
   `products`, `users`, `auth` — sem alterar as rotas existentes.
6. **`prisma/schema.prisma`**: novas models (`WebhookEndpoint`, `WebhookOutbox`,
   `WebhookDeadLetter`, `WebhookDelivery`) seguem a convenção já usada em `Order`/
   `OrderItem`/`OrderStatusHistory`: PK `String @id @default(uuid()) @db.Char(36)`,
   `@@map` em snake_case, índices em `status`/`createdAt`/chaves de filtro. Não há
   alteração em nenhuma model existente.

## Critérios de aceite técnicos

- Uma mudança de status que não tenha webhook inscrito para aquele `toStatus` não gera
  nenhuma linha na outbox (evita ruído).
- Uma mudança de status com webhook inscrito gera exatamente 1 linha na outbox por
  endpoint elegível, dentro da mesma transação de `changeStatus`; falha na inserção
  reverte a mudança de status.
- Entrega bem-sucedida (2xx dentro de 10s) marca o evento como `delivered` e registra
  em `webhook_delivery`.
- Falha de entrega segue a progressão de backoff definida e, após a 5ª falha, o evento
  aparece em `webhook_dead_letter` — nunca é descartado silenciosamente.
- Toda entrega ao cliente carrega os headers `X-Event-Id`, `X-Signature`,
  `X-Timestamp`, `X-Webhook-Id` e o payload é validável via HMAC-SHA256 com a secret
  do endpoint.
- Replay de DLQ só é aceito para usuários com role `ADMIN` e é registrado em log com o
  autor da ação.

## Riscos e mitigação

| Risco | Mitigação |
| --- | --- |
| Escrita na outbox aumenta o tempo/complexidade da transação `changeStatus` | Inserção local, sem chamada de rede, dentro do mesmo banco; índice em `status`/`created_at` mantém a leitura do worker eficiente mesmo com volume |
| Worker cai e para de processar a outbox | Processo separado, reiniciável independentemente da API (ver [ADR-002](./adrs/ADR-002-worker-separado-com-polling.md)); eventos ficam retidos como `pending` até o worker voltar, sem perda |
| Vazamento de secret do lado do cliente | Secret por endpoint (não global) + rotação com grace period de 24h (ver [ADR-004](./adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)) |
| Cliente recebe evento duplicado | Contrato explícito de at-least-once + `X-Event-Id` para dedup (ver [ADR-005](./adrs/ADR-005-garantia-at-least-once-com-x-event-id.md)); documentação clara no portal do desenvolvedor |
