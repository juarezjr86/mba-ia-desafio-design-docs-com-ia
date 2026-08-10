# Tracker de Rastreabilidade

Mapeia cada item relevante de `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` e
`docs/adrs/*.md` à sua origem em `TRANSCRICAO.md` (formato `[hh:mm] Nome`) ou no
código-fonte (caminho real de arquivo). Linhas sem origem verificável não entram nesta
tabela — nesse caso, o item correspondente é corrigido ou removido do documento de
origem (regra de `CLAUDE.md`).

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-OBJ-01 | docs/PRD.md | Requisito Não Funcional | Latência-alvo <10s = definição de "tempo real" dos clientes | TRANSCRICAO | [09:02] Marcos |
| PRD-ESCOPO-01 | docs/PRD.md | Restrição (fora de escopo) | E-mail de alerta ao cliente após falhas — adiado (pergunta de Marcos, decisão de Larissa) | TRANSCRICAO | [09:37] Larissa |
| PRD-ESCOPO-02 | docs/PRD.md | Restrição (fora de escopo) | Dashboard visual para o cliente — fora de escopo (pergunta de Marcos, decisão de Larissa) | TRANSCRICAO | [09:40] Larissa |
| PRD-ESCOPO-03 | docs/PRD.md | Restrição (fora de escopo) | Rate limiting de envio — "observar e decidir depois" (tema levantado por Diego, decisão de Larissa) | TRANSCRICAO | [09:39] Larissa |
| PRD-ESCOPO-04 | docs/PRD.md | Restrição (fora de escopo) | Múltiplos workers em paralelo — adiado | TRANSCRICAO | [09:13] Diego |
| PRD-ESCOPO-05 | docs/PRD.md | Restrição (fora de escopo) | Garantia de ordering global — não exigida | TRANSCRICAO | [09:14] Marcos |
| PRD-ESCOPO-06 | docs/PRD.md | Restrição (fora de escopo) | Arquivamento de eventos entregues — fora de escopo | TRANSCRICAO | [09:08] Diego |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro de webhook (URL, status, secret gerada pelo servidor) | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Recusa de URL não-HTTPS no cadastro | TRANSCRICAO | [09:23] Sofia |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Edição (PATCH) de webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Remoção (DELETE) de webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Listagem (GET) de webhooks do customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Filtragem de evento por status inscrito, na inserção (pergunta de Diego, formulação de Bruno) | TRANSCRICAO | [09:34] Bruno |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Assinatura HMAC-SHA256 do payload | TRANSCRICAO | [09:20] Sofia |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Retry com backoff até 5 tentativas antes de DLQ | TRANSCRICAO | [09:15] Diego |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Replay manual de DLQ por ADMIN, com auditoria | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | `X-Event-Id` para deduplicação no cliente | TRANSCRICAO | [09:25] Diego |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s por chamada HTTP do worker | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Limite de 64KB no payload, erro se exceder | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | HTTPS obrigatório na comunicação de webhook | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Garantia at-least-once (nunca exactly-once) | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Feature não pode degradar robustez de `changeStatus` | TRANSCRICAO | [09:40] Bruno |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Reuso de Pino e `AppError`/error middleware, sem ferramenta nova | TRANSCRICAO | [09:29] Bruno |
| PRD-RISK-01 | docs/PRD.md | Risco | Vazamento de secret já ocorrido com outro cliente | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-02 | docs/PRD.md | Risco | Indisponibilidade de cliente de até 2h já observada | TRANSCRICAO | [09:16] Diego |
| PRD-RISK-03 | docs/PRD.md | Risco | Cliente pode falhar em deduplicar corretamente | TRANSCRICAO | [09:25] Sofia |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança da Sofia, ≥2 dias úteis, antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Comunicação aos clientes via portal de desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| RFC-META-01 | docs/RFC.md | Restrição | Autoria do RFC atribuída à Larissa | TRANSCRICAO | [09:50] Larissa |
| RFC-PROP-01 | docs/RFC.md | Decisão | Emissão desacoplada via Outbox, função `publishWebhookEvent` | TRANSCRICAO | [09:41] Bruno |
| RFC-PROP-02 | docs/RFC.md | Decisão | Entrega via worker dedicado, polling 2s | TRANSCRICAO | [09:09] Diego |
| RFC-PROP-03 | docs/RFC.md | Decisão | Retry 5x, backoff 1m/5m/30m/2h/12h, DLQ | TRANSCRICAO | [09:17] Diego |
| RFC-PROP-04 | docs/RFC.md | Decisão | HMAC-SHA256, secret por endpoint, rotação 24h | TRANSCRICAO | [09:22] Sofia |
| RFC-PROP-05 | docs/RFC.md | Decisão | At-least-once + `X-Event-Id` | TRANSCRICAO | [09:26] Larissa |
| RFC-PROP-06 | docs/RFC.md | Decisão | Módulo `src/modules/webhooks/` no padrão existente | TRANSCRICAO | [09:27] Bruno |
| RFC-ALT-01 | docs/RFC.md | Trade-off | Síncrono descartado — trava mudança de status de outros pedidos | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Fila externa (Redis Streams) descartada — overengineering | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Trade-off | Retry indefinido descartado — evento pendurado para sempre | TRANSCRICAO | [09:15] Diego |
| RFC-ALT-04 | docs/RFC.md | Trade-off | Exactly-once descartado — complexidade desproporcional | TRANSCRICAO | [09:25] Diego |
| RFC-OPEN-01 | docs/RFC.md | Questão em aberto | Rate limiting de envio — não decidido | TRANSCRICAO | [09:39] Larissa |
| RFC-OPEN-02 | docs/RFC.md | Questão em aberto | Notificação proativa de falha (e-mail) — adiada | TRANSCRICAO | [09:38] Marcos |
| RFC-OPEN-03 | docs/RFC.md | Questão em aberto | Escalonamento para múltiplos workers — não decidido | TRANSCRICAO | [09:13] Diego |
| RFC-IMPACT-01 | docs/RFC.md | Restrição | Escrita adicional na transação `changeStatus` | CODIGO | src/modules/orders/order.service.ts |
| RFC-IMPACT-02 | docs/RFC.md | Risco | Novo processo (worker) exige monitoramento próprio | TRANSCRICAO | [09:11] Diego |
| RFC-IMPACT-03 | docs/RFC.md | Risco | Exposição de dados de pedido a sistemas externos | TRANSCRICAO | [09:19] Sofia |
| RFC-IMPACT-04 | docs/RFC.md | Restrição | Limitação de ordering só por order_id, single-worker | TRANSCRICAO | [09:13] Larissa |
| ADR-001 | docs/adrs/ADR-001-outbox-pattern-no-mysql.md | Decisão | Outbox Pattern no MySQL | TRANSCRICAO | [09:06] Diego |
| ADR-001-ALT | docs/adrs/ADR-001-outbox-pattern-no-mysql.md | Trade-off | Síncrono e fila externa descartados | TRANSCRICAO | [09:04][09:07] Bruno, Diego |
| ADR-001-CODE | docs/adrs/ADR-001-outbox-pattern-no-mysql.md | Decisão | Inserção na mesma transação de `changeStatus` | CODIGO | src/modules/orders/order.service.ts |
| ADR-002 | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisão | Worker separado, polling de 2s | TRANSCRICAO | [09:09] Diego |
| ADR-002-ALT | docs/adrs/ADR-002-worker-separado-com-polling.md | Trade-off | Trigger nativo do MySQL descartado | TRANSCRICAO | [09:09] Diego |
| ADR-002-CODE | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisão | `src/worker.ts` como entry-point análoga | CODIGO | src/server.ts |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | Retry 5x com backoff e DLQ separada | TRANSCRICAO | [09:15][09:18] Diego |
| ADR-003-ALT | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Trade-off | 3 tentativas descartadas — indisponibilidade de 2h observada | TRANSCRICAO | [09:16] Bruno, Diego |
| ADR-003-CODE | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | Replay de DLQ exige `requireRole('ADMIN')` | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256, secret por endpoint, rotação 24h | TRANSCRICAO | [09:20][09:21] Sofia |
| ADR-004-ALT | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Trade-off | Secret global descartada — vaza tudo se comprometida | TRANSCRICAO | [09:21] Sofia |
| ADR-005 | docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md | Decisão | At-least-once com dedup via `X-Event-Id` | TRANSCRICAO | [09:25] Diego |
| ADR-005-ALT | docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md | Trade-off | Exactly-once descartado — padrão de mercado citado (Stripe/GitHub) | TRANSCRICAO | [09:25] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Reuso do padrão modular e de infraestrutura compartilhada | TRANSCRICAO | [09:27] Bruno |
| ADR-006-CODE | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Novas classes de erro estendem `AppError` | CODIGO | src/shared/errors/app-error.ts |
| FDD-FLUXO-01 | docs/FDD.md | Requisito Funcional | Criação do evento na outbox dentro da transação | TRANSCRICAO | [09:41] Bruno |
| FDD-FLUXO-01-CODE | docs/FDD.md | Requisito Funcional | Ponto de chamada: `changeStatus` | CODIGO | src/modules/orders/order.service.ts |
| FDD-FLUXO-02 | docs/FDD.md | Requisito Funcional | Processamento pelo worker, lote pequeno, ordenado por created_at | TRANSCRICAO | [09:08][09:09] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Requisito Funcional | Retry com backoff por tentativa | TRANSCRICAO | [09:15][09:17] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Requisito Funcional | Movimentação para DLQ após 5ª falha e replay administrativo | TRANSCRICAO | [09:18][09:35] Diego, Larissa |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | `POST /webhooks` — secret devolvida só na criação | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | `GET /webhooks` — listagem paginada | CODIGO | src/shared/http/response.ts |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | `PATCH /webhooks/:id` | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | `GET /webhooks/:id/deliveries` — últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | `POST /admin/webhooks/dead-letter/:id/replay` | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | Payload de evento: campos e exclusão de `items` | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | Headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` | TRANSCRICAO | [09:44] Diego, Sofia |
| FDD-CONTRATO-08 | docs/FDD.md | Hipótese | Secret não retornada em GET/listagem — inferência de segurança, não decidida na reunião | Não informado (marcado como hipótese no próprio FDD) | — |
| FDD-ERRO-01 | docs/FDD.md | Restrição | Código `WEBHOOK_NOT_FOUND` | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-02 | docs/FDD.md | Restrição | `WEBHOOK_INVALID_URL` por URL não-HTTPS | TRANSCRICAO | [09:23] Sofia |
| FDD-ERRO-03 | docs/FDD.md | Restrição | Comportamento de erro acima de 64KB (nome `WEBHOOK_PAYLOAD_TOO_LARGE` é inferência, não citado em ata) | TRANSCRICAO | [09:23][09:24] Sofia, Diego |
| FDD-ERRO-04 | docs/FDD.md | Restrição | Comportamento de timeout acima de 10s (nome `WEBHOOK_DELIVERY_TIMEOUT` é inferência, não citado em ata) | TRANSCRICAO | [09:42] Diego |
| FDD-ERRO-05 | docs/FDD.md | Restrição | Prefixo `WEBHOOK_` seguindo padrão de código existente | TRANSCRICAO | [09:28][09:29] Bruno, Larissa |
| FDD-ERRO-06 | docs/FDD.md | Restrição | Reuso de `VALIDATION_ERROR`/`UNAUTHORIZED`/`FORBIDDEN`/`NOT_FOUND` | CODIGO | src/shared/errors/http-errors.ts |
| FDD-RESILIENCIA-01 | docs/FDD.md | Requisito Não Funcional | Timeout de 10s | TRANSCRICAO | [09:42] Diego |
| FDD-RESILIENCIA-02 | docs/FDD.md | Requisito Não Funcional | Backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| FDD-RESILIENCIA-03 | docs/FDD.md | Restrição | Sem fallback de canal (e-mail) nesta fase | TRANSCRICAO | [09:37] Larissa |
| FDD-OBS-01 | docs/FDD.md | Requisito Não Funcional | Reuso da instância Pino existente | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-02 | docs/FDD.md | Requisito Não Funcional | `event_id` como identificador de correlação ponta a ponta | TRANSCRICAO | [09:25] Diego |
| FDD-OBS-03 | docs/FDD.md | Requisito Não Funcional | Padrão de correlação `X-Request-Id`/`req.id` reaproveitado | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-INTEGRACAO-01 | docs/FDD.md | Restrição | Integração com `changeStatus` | CODIGO | src/modules/orders/order.service.ts |
| FDD-INTEGRACAO-02 | docs/FDD.md | Restrição | Novas classes de erro estendem `AppError`/http-errors | CODIGO | src/shared/errors/app-error.ts |
| FDD-INTEGRACAO-03 | docs/FDD.md | Restrição | Captura automática pelo error middleware | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INTEGRACAO-04 | docs/FDD.md | Restrição | `authenticate`/`requireRole` reaproveitados | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INTEGRACAO-05 | docs/FDD.md | Restrição | Novo router agregado ao roteador principal | CODIGO | src/routes/index.ts |
| FDD-INTEGRACAO-06 | docs/FDD.md | Restrição | Novas models seguem convenção UUID/`@@map` | CODIGO | prisma/schema.prisma |
| FDD-CRITERIO-01 | docs/FDD.md | Critério de aceite | Sem evento gerado se nenhum webhook inscrito no status | TRANSCRICAO | [09:34] Bruno |
| FDD-CRITERIO-02 | docs/FDD.md | Critério de aceite | Falha na inserção reverte a mudança de status | TRANSCRICAO | [09:40][09:41] Bruno, Diego |
| FDD-CRITERIO-03 | docs/FDD.md | Critério de aceite | Replay de DLQ só para role ADMIN, com log de autor | TRANSCRICAO | [09:36] Sofia |
| FDD-RISK-01 | docs/FDD.md | Risco | Worker fora do ar retém eventos como pending, sem perda | TRANSCRICAO | [09:11] Diego |
| FDD-RISK-02 | docs/FDD.md | Risco | Estoque/status usa mesma máquina de estados já existente | CODIGO | src/modules/orders/order.status.ts |

## Cobertura

Contagem exaustiva (verificada por busca automatizada na tabela acima, não por
estimativa):

- Total de linhas: 99 (a contagem anterior desta seção dizia "100" — estava errada;
  recontada em 2ª auditoria por revisor independente, ver `README.md` iteração 7).
- Fonte `CODIGO`: 17 linhas (≥5 exigido).
- Fonte `TRANSCRICAO`: 81 linhas com timestamp válido (81% do total, ≥70% exigido).
- 1 linha (`FDD-CONTRATO-08`) marcada como "Não informado" — hipótese/inferência sem
  origem em transcrição ou código — mantida no tracker exatamente para deixar essa
  lacuna visível, não para escondê-la. Não conta nem como `CODIGO` nem como
  `TRANSCRICAO`.
