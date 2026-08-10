# ADR-006 — Reuso máximo dos padrões existentes do projeto

## Status

Aceito — decidido em reunião técnica (data exata não informada em `TRANSCRICAO.md`, que só registra "quinta-feira, 09:00"; ver [09:27]–[09:30]).

## Contexto

Bruno propôs alinhar a implementação da feature de webhooks à estrutura já existente
no código do OMS em vez de introduzir convenções novas, para reduzir a curva de
aprendizado do time e manter consistência arquitetural
(`TRANSCRICAO.md` [09:27]–[09:30] Bruno, Diego, Larissa).

## Decisão

O módulo de webhooks segue **exatamente** o padrão modular já usado em
`src/modules/orders`, `src/modules/customers`, `src/modules/products`: pasta
`src/modules/webhooks/` com `webhook.controller.ts`, `webhook.service.ts`,
`webhook.repository.ts`, `webhook.routes.ts`, `webhook.schemas.ts`
(`TRANSCRICAO.md` [09:27] Bruno). O worker é uma entry-point separada
(`src/worker.ts`), com a lógica de processamento dentro do módulo
(`webhook.worker.ts`/`webhook.processor.ts`) (`TRANSCRICAO.md` [09:28] Bruno).

Reuso explícito de infraestrutura compartilhada já existente, sem criar nada
paralelo:

- **Erros**: novas classes estendendo `AppError`
  (`src/shared/errors/app-error.ts`), com códigos prefixados `WEBHOOK_*`
  (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`),
  seguindo o padrão de `InsufficientStockError`/`InvalidStatusTransitionError`
  (`TRANSCRICAO.md` [09:28][09:29] Bruno, Larissa).
- **Middleware de erro**: `src/middlewares/error.middleware.ts` já trata qualquer
  `AppError` — nenhuma alteração necessária para capturar os novos erros do módulo
  (`TRANSCRICAO.md` [09:29] Bruno).
- **Logger**: instância Pino já existente em `src/shared/logger/index.ts`, sem novo
  transporte ou biblioteca (`TRANSCRICAO.md` [09:29] Bruno).
- **Autenticação/autorização**: `authenticate` e `requireRole` de
  `src/middlewares/auth.middleware.ts`, sem novo mecanismo de auth.
- **Persistência**: mesmo `PrismaClient`/MySQL, seguindo as convenções de
  `prisma/schema.prisma` (UUID como chave primária, `@@map` em snake_case).

## Alternativas Consideradas

- **Criar uma estrutura própria para o módulo de webhooks** (ex.: camadas diferentes,
  biblioteca de log separada, mecanismo de erro próprio). Não foi proposta por
  nenhum participante da reunião; foi implicitamente descartada quando Bruno afirma
  "a gente tem um padrão claro na codebase [...] webhook vai seguir igual" e ninguém
  discorda (`TRANSCRICAO.md` [09:27]–[09:30]).

## Consequências

**Positivas**
- Curva de aprendizado baixa para qualquer desenvolvedor do time que já conhece os
  módulos existentes (`orders`, `customers`, `products`).
- Reduz risco de bugs por reinvenção: tratamento de erro, logging e autenticação já
  são testados e usados em produção nos módulos existentes.
- Facilita revisão de código e revisão de segurança (Sofia já conhece o formato do
  middleware de erro e do padrão de auth).

**Negativas**
- Herda quaisquer limitações do padrão atual (ex.: composição manual de dependências
  em `src/app.ts`, sem um container de injeção de dependência) — qualquer refatoração
  futura da arquitetura base afeta também o módulo de webhooks.
- Menos liberdade para escolher uma estrutura eventualmente mais adequada às
  particularidades de um sistema orientado a eventos (ex.: um padrão mais explícito de
  publish/subscribe), em nome da consistência com o restante do projeto.
