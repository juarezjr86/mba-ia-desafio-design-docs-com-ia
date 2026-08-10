# Architectural Decision Records

ADRs da feature de Webhooks de Notificação de Pedidos, no formato MADR, nomeados
`ADR-NNN-titulo-em-kebab-case.md`. Toda decisão aqui tem origem em
`../../TRANSCRICAO.md` (ver `../TRACKER.md` para o mapeamento).

| ADR | Decisão |
| --- | --- |
| [ADR-001](./ADR-001-outbox-pattern-no-mysql.md) | Outbox Pattern no MySQL para emissão de eventos |
| [ADR-002](./ADR-002-worker-separado-com-polling.md) | Worker em processo separado, polling de 2s |
| [ADR-003](./ADR-003-retry-com-backoff-exponencial-e-dlq.md) | Retry com backoff exponencial (5 tentativas) e DLQ |
| [ADR-004](./ADR-004-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint e rotação |
| [ADR-005](./ADR-005-garantia-at-least-once-com-x-event-id.md) | Garantia at-least-once com dedup via `X-Event-Id` |
| [ADR-006](./ADR-006-reuso-dos-padroes-existentes-do-projeto.md) | Reuso máximo dos padrões existentes do projeto |
