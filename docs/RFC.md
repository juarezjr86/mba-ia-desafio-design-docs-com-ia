# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
| --- | --- |
| Autor | Larissa (Tech Lead) — "Eu vou abrir o doc de design da feature" (`TRANSCRICAO.md` [09:50] Larissa) |
| Status | Em revisão |
| Data | Não informada em `TRANSCRICAO.md` (cabeçalho registra apenas "quinta-feira, 09:00", sem data completa) |
| Revisores | Marcos (PM), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Eng. Segurança) |

## Resumo executivo (TL;DR)

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) pediram notificação
em tempo real (<10s) de mudança de status de pedido, hoje resolvida via polling manual
no `GET /orders` (`TRANSCRICAO.md` [09:00][09:02] Marcos). Propomos um sistema de
webhooks **outbound** (o OMS envia, não recebe) baseado no padrão **Transactional
Outbox** sobre o MySQL já usado pelo projeto: a mudança de status insere um evento na
tabela `webhook_outbox` na mesma transação de `changeStatus`; um **worker em processo
separado**, em polling de 2s, lê a outbox e entrega os eventos via HTTP, com retry
exponencial, DLQ, assinatura HMAC-SHA256 e garantia at-least-once. Não introduzimos
nenhuma peça de infraestrutura nova (sem fila externa, sem broker) — tudo roda sobre o
MySQL e o Node.js já usados pelo OMS.

## Contexto

O OMS não tem, hoje, nenhum mecanismo de notificação externa, evento ou fila. Os
clientes B2B ficam repetindo `GET /orders` para detectar mudança de status, o que a
Atlas descreveu como lento e caro de manter do lado deles — a ponto de ameaçar migrar
para um concorrente se a feature não sair até o fim do trimestre
(`TRANSCRICAO.md` [09:00] Marcos). Este RFC propõe a solução técnica discutida e
fechada em reunião entre Tech Lead, PM, Engenharia (Pedidos e Plataforma) e Segurança.

## Problema

Como notificar sistemas de clientes externos sobre mudança de status de pedido, com
latência abaixo de 10 segundos, sem comprometer a robustez da transação de negócio já
existente (`changeStatus` em `src/modules/orders/order.service.ts`) e sem depender da
disponibilidade dos sistemas dos clientes para que o OMS continue funcionando
normalmente?

## Proposta técnica

1. **Emissão desacoplada via Outbox** (ver [ADR-001](./adrs/ADR-001-outbox-pattern-no-mysql.md)).
   Dentro da mesma transação de `changeStatus`, uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)`
   insere um evento (payload já renderizado/snapshot no momento da mudança) na tabela
   `webhook_outbox`, filtrando apenas os webhooks do customer que estão inscritos
   naquele status (`TRANSCRICAO.md` [09:33][09:34][09:41][09:52]). Se a inserção falhar,
   a transação inteira sofre rollback — não existe caso de status mudar sem o evento
   correspondente ser registrado.
2. **Entrega via worker dedicado** (ver [ADR-002](./adrs/ADR-002-worker-separado-com-polling.md)).
   Processo Node separado (`src/worker.ts`), polling a cada 2s, lê eventos pendentes
   por ordem de `created_at`, chama o endpoint HTTPS cadastrado do cliente com timeout
   de 10s.
3. **Resiliência de entrega** (ver [ADR-003](./adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)).
   5 tentativas com backoff 1m/5m/30m/2h/12h; esgotadas, o evento vai para
   `webhook_dead_letter`, reprocessável manualmente via
   `POST /admin/webhooks/dead-letter/:id/replay` (role `ADMIN`).
4. **Segurança da entrega** (ver [ADR-004](./adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)).
   HMAC-SHA256 sobre o corpo, secret única por endpoint, rotação com grace period de
   24h, HTTPS obrigatório no cadastro, payload limitado a 64KB.
5. **Contrato de entrega** (ver [ADR-005](./adrs/ADR-005-garantia-at-least-once-com-x-event-id.md)).
   Garantia at-least-once; `X-Event-Id` (UUID) para deduplicação do lado do cliente.
6. **Estrutura de código** (ver [ADR-006](./adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)).
   Módulo `src/modules/webhooks/` no mesmo padrão dos módulos existentes; erros
   `WEBHOOK_*` estendendo `AppError`; reuso de Pino, `error.middleware.ts` e
   `auth.middleware.ts` sem modificação.

Superfície de API proposta (detalhamento completo de payloads e status codes fica no
[FDD](./FDD.md)): CRUD de configuração de webhook (`POST/GET/PATCH/DELETE /webhooks`),
consulta de histórico de entregas (`GET /webhooks/:id/deliveries`) e replay
administrativo de DLQ.

## Alternativas consideradas

| Alternativa | Trade-off que motivou o descarte |
| --- | --- |
| Disparo HTTP síncrono dentro de `changeStatus` | Acopla a latência/disponibilidade de sistemas de clientes externos à transação crítica de mudança de status; sem estratégia viável de rollback se o cliente estiver fora do ar (`TRANSCRICAO.md` [09:04] Bruno) |
| Fila externa dedicada (ex.: Redis Streams) | Resolveria o desacoplamento, mas exigiria subir e operar infraestrutura nova — considerado overengineering para o tamanho do time e do problema (`TRANSCRICAO.md` [09:07] Diego, Larissa) |
| Retry indefinido, sem teto de tentativas | Evita perder eventos definitivamente, mas mantém eventos "pendurados" para sempre quando o cliente já não existe mais, sem sinal claro de falha permanente (`TRANSCRICAO.md` [09:15] Diego) |
| Garantia exactly-once na entrega | Eliminaria a necessidade de dedup do lado do cliente, mas exigiria coordenação transacional entre os dois sistemas — complexidade desproporcional ao ganho; at-least-once + `X-Event-Id` já resolve a maior parte dos casos e é o padrão adotado por provedores de referência (`TRANSCRICAO.md` [09:25] Diego) |

## Questões em aberto

- **Rate limiting de envio ao cliente.** Se um cliente tiver muitos pedidos mudando de
  status em pouco tempo, o worker pode disparar um burst de chamadas HTTP para ele.
  Diego levantou o ponto; a decisão da reunião foi "observar e decidir depois" — não
  entra no escopo desta primeira entrega (`TRANSCRICAO.md` [09:38][09:39] Diego,
  Larissa).
- **Notificação proativa de falhas ao cliente** (ex.: e-mail após N falhas
  consecutivas). Fora de escopo desta fase; considerada para uma fase futura, após medir
  o impacto real (`TRANSCRICAO.md` [09:37][09:38] Marcos, Larissa).
- **Escalonamento para múltiplos workers em paralelo.** Não decidido nesta reunião;
  exigiria particionamento por `order_id` ou lock pessimista para preservar ordering,
  e foi explicitamente adiado como "problema do futuro" (`TRANSCRICAO.md` [09:13]
  Diego, Bruno).

## Impacto e riscos

- **Impacto no fluxo crítico existente:** `OrderService.changeStatus` passa a incluir
  uma escrita adicional (na outbox) dentro da mesma transação — qualquer falha nessa
  escrita agora também reverte a mudança de status. Mitigação: a escrita na outbox é uma
  inserção simples e local (mesmo banco, mesma transação), sem chamada de rede.
- **Novo processo em produção** (worker) precisa de monitoramento e restart
  automático próprios — risco operacional novo que a API sozinha não tinha.
- **Exposição de dados de pedido a sistemas externos** — mitigado por HMAC-SHA256,
  HTTPS obrigatório e secret por endpoint (ver ADR-004); Sofia reservou pelo menos 2
  dias úteis de revisão de segurança antes do deploy, focada em HMAC e geração de
  secret (`TRANSCRICAO.md` [09:46] Sofia).
- **Limitação de ordering:** eventos são ordenados apenas por `order_id`, e apenas
  enquanto houver um único worker; não há garantia de ordering global entre pedidos
  diferentes. Os clientes não pediram essa garantia (`TRANSCRICAO.md` [09:12]–[09:14]
  Diego, Marcos).

## Decisões relacionadas

- [ADR-001 — Outbox Pattern no MySQL](./adrs/ADR-001-outbox-pattern-no-mysql.md)
- [ADR-002 — Worker separado com polling](./adrs/ADR-002-worker-separado-com-polling.md)
- [ADR-003 — Retry com backoff exponencial e DLQ](./adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)
- [ADR-004 — HMAC-SHA256 com secret por endpoint](./adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)
- [ADR-005 — Garantia at-least-once com X-Event-Id](./adrs/ADR-005-garantia-at-least-once-com-x-event-id.md)
- [ADR-006 — Reuso dos padrões existentes do projeto](./adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)

Detalhamento de implementação (contratos HTTP, fluxos passo a passo, matriz de erros,
observabilidade): ver [FDD](./FDD.md).
