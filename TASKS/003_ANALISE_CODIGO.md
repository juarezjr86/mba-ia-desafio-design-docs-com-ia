# TASK 003 — Análise do Código

## Objetivo

Consolidar os pontos de integração reais entre a feature de Webhooks e o código
existente, com caminho de arquivo confirmado, para uso no FDD e nos ADRs.

## Contexto

Complementa `TASKS/001_EXPLORACAO.md` com foco específico em "onde a feature nova
encosta no código antigo".

## Entregáveis

Tabela de pontos de integração (abaixo).

## Status

🟢 Concluído.

## Itens concluídos

| Ponto de integração | Arquivo real | Como a feature usa |
| --- | --- | --- |
| Transação de mudança de status | `src/modules/orders/order.service.ts` (método `changeStatus`) | Inserir evento na outbox dentro do mesmo `this.prisma.$transaction`, via função `publishWebhookEvent(tx, order, from, to)` conforme decidido em [09:41] |
| Máquina de estados | `src/modules/orders/order.status.ts` | Fonte de `from_status`/`to_status` válidos para o payload do evento; nenhuma alteração necessária, só leitura |
| Erros de domínio | `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts` | Novas classes de erro do módulo webhooks estendem `AppError`, seguindo o mesmo padrão de `InvalidStatusTransitionError`/`InsufficientStockError`, com códigos prefixados `WEBHOOK_*` |
| Middleware de erro centralizado | `src/middlewares/error.middleware.ts` | Já trata qualquer `AppError`; nenhuma mudança necessária para as novas classes de erro do módulo webhooks serem capturadas |
| Autenticação e autorização | `src/middlewares/auth.middleware.ts` (`authenticate`, `requireRole`) | CRUD de webhook usa `authenticate` normal; endpoint de replay de DLQ usa `requireRole('ADMIN')` |
| Logger estruturado | `src/shared/logger/index.ts` | Worker e módulo webhooks reusam a instância `logger` (Pino) existente, sem novo transporte |
| Padrão de módulo | `src/modules/orders/*` (controller/service/repository/routes/schemas) | Modelo estrutural replicado em `src/modules/webhooks/*` |
| Bootstrap de processo | `src/server.ts` | Padrão de entry-point replicado para novo `src/worker.ts` (novo processo, não editar `server.ts`) |
| Composição de dependências | `src/app.ts` (`buildControllers`) | Novo controller de webhooks se registra no mesmo padrão de composição manual (sem framework de DI) |
| Roteamento agregado | `src/routes/index.ts` | Novo router de webhooks se agrega via `router.use('/webhooks', ...)` no mesmo padrão dos demais módulos |
| Persistência / schema | `prisma/schema.prisma` | Novas tabelas (`webhook_endpoint`, `webhook_outbox`, `webhook_dead_letter`, `webhook_delivery`) seguem convenção existente: PK `String @id @default(uuid()) @db.Char(36)`, `@@map` em snake_case, índices em campos de filtro/status |
| Padrão de resposta paginada | `src/shared/http/response.ts` (`paginated`) | Reuso para `GET /webhooks` e `GET /webhooks/:id/deliveries` |
| Validação de schema | `src/modules/orders/order.schemas.ts` (padrão Zod) | Modelo para os schemas Zod do módulo webhooks (ex: `.url()` + refine para exigir `https`) |
| Config de ambiente | `src/config/env.ts` | Nenhuma variável nova referenciada na transcrição; qualquer nova env var (ex: intervalo de polling) deve ser tratada como inferência/hipótese no FDD, não decisão fechada |
| Teste de referência | `tests/orders.test.ts`, `tests/helpers/factories.ts` | Modelo de estilo de teste (Vitest + Supertest + app real) a citar na Estratégia de Testes do PRD/FDD — sem criar teste algum, já que `tests/` não deve ser alterado |

## Riscos

Nenhum além dos já registrados em TASKS/002.

## Próximos passos

Iniciar produção de ADRs (`TASKS/004_ADRS.md`).

## Observações

Todos os caminhos foram confirmados via Read direto dos arquivos nesta sessão.
