# TASK 001 — Exploração do Projeto

## Objetivo

Mapear a estrutura do repositório, tecnologias, arquitetura e convenções antes de
qualquer produção documental.

## Contexto

Etapa 1 do fluxo obrigatório definido em `CLAUDE.md`. Nada foi documentado antes desta
exploração; o repositório clonado contém apenas o boilerplate do desafio.

## Entregáveis

Mapa de código (resumo abaixo) — insumo para `TASKS/003_ANALISE_CODIGO.md` e para as
seções "Integração com o sistema existente" do FDD.

## Status

🟢 Concluído.

## Dependências

Nenhuma.

## Itens pendentes

Nenhum.

## Itens concluídos

- Linguagem/stack: Node.js 20+, TypeScript, Express 4, Prisma 5 (MySQL 8), Zod, JWT
  (`jsonwebtoken`), bcrypt, Pino (+ pino-http), Vitest + Supertest para testes.
- Arquitetura: modular por domínio em `src/modules/<dominio>/`, cada módulo com
  `*.controller.ts`, `*.service.ts`, `*.repository.ts`, `*.routes.ts`, `*.schemas.ts`.
  Composição em `src/app.ts` (`buildControllers`, `buildApp`), roteamento agregado em
  `src/routes/index.ts`, bootstrap em `src/server.ts`.
- Domínios existentes: `auth`, `users`, `customers`, `products`, `orders`.
- Máquina de estados de pedido: `src/modules/orders/order.status.ts` — transições
  válidas `PENDING→{PAID,CANCELLED}`, `PAID→{PROCESSING,CANCELLED}`,
  `PROCESSING→{SHIPPED,CANCELLED}`, `SHIPPED→{DELIVERED}`, `DELIVERED`/`CANCELLED`
  terminais. Débito de estoque só em `PENDING→PAID`; reposição em `{PAID,PROCESSING}→CANCELLED`.
- Transação crítica: `src/modules/orders/order.service.ts#changeStatus` — dentro de
  `this.prisma.$transaction`, valida transição, debita/repõe estoque, atualiza `order`,
  insere em `orderStatusHistory`. É o ponto de integração central da feature de
  webhooks (inserir na outbox dentro dessa mesma transação).
- Erros: `src/shared/errors/app-error.ts` (`AppError` base: message, statusCode,
  errorCode, details) e `src/shared/errors/http-errors.ts` (subclasses como
  `NotFoundError`, `ConflictError`, `UnprocessableEntityError`,
  `InvalidStatusTransitionError`, `InsufficientStockError`). Padrão de código:
  `SCREAMING_SNAKE_CASE` (ex: `INVALID_STATUS_TRANSITION`, `INSUFFICIENT_STOCK`).
- Error middleware centralizado: `src/middlewares/error.middleware.ts` — trata
  `AppError`, `ZodError`, `Prisma.PrismaClientKnownRequestError` (P2002, P2025) e
  fallback 500 com log via Pino.
- Auth: `src/middlewares/auth.middleware.ts` — `authenticate` (JWT bearer,
  `AuthUser { id, email, role: 'ADMIN' | 'OPERATOR' }`) e `requireRole(...roles)`.
- Logger: `src/shared/logger/index.ts` — Pino, redact de campos sensíveis, pretty
  print em dev.
- Request logging: `src/middlewares/request-logger.middleware.ts` — `X-Request-Id`,
  duração, método, path, status, userId.
- Persistência: `prisma/schema.prisma` — models `User`, `Customer`, `Product`,
  `Order`, `OrderItem`, `OrderStatusHistory`, `OrderNumberSequence`. Todos os IDs são
  UUID (`@default(uuid())`, `@db.Char(36)`). MySQL 8 via `docker-compose.yml`.
- Testes: `tests/orders.test.ts`, `tests/auth.test.ts`, `tests/helpers/factories.ts`
  (Vitest + Supertest, app real via `getTestApp()`).
- Não existe, hoje, nenhum mecanismo de notificação externa, eventos, filas ou
  webhooks — confirmado por ausência de qualquer módulo/tabela correspondente.

## Riscos

Nenhum específico desta etapa.

## Próximos passos

`TASKS/002_ANALISE_TRANSCRICAO.md`.

## Observações

Todos os caminhos acima foram confirmados via leitura direta do arquivo (Read tool),
não inferidos.
