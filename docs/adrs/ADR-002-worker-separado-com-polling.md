# ADR-002 — Worker em processo separado, com polling de 2 segundos

## Status

Aceito — decidido em reunião técnica (data exata não informada em `TRANSCRICAO.md`, que só registra "quinta-feira, 09:00"; ver [09:09]–[09:13],
[09:28]).

## Contexto

Definido o padrão Outbox (ADR-001), era preciso decidir como e onde os eventos
pendentes em `webhook_outbox` são lidos e entregues. Diego levantou que o worker
precisa rodar como **processo separado** da API, e não dentro da mesma instância —
senão a reinicialização da API derrubaria o worker junto (`TRANSCRICAO.md` [09:11]
Diego). Bruno perguntou se seria possível usar trigger nativo do MySQL para reagir de
forma mais reativa (`TRANSCRICAO.md` [09:09] Bruno).

## Decisão

O worker é um **processo Node.js separado**, com entry-point próprio
(`src/worker.ts`, seguindo o padrão de `src/server.ts`, e script `npm run worker`,
conforme proposto por Larissa em [09:11]). A lógica de processamento fica dentro do
módulo webhooks (`src/modules/webhooks/webhook.worker.ts` ou `webhook.processor.ts`,
conforme Bruno em [09:28]).

O worker faz **polling em loop a cada 2 segundos**: busca os eventos pendentes mais
antigos em `webhook_outbox`, processa em lote pequeno, marca como entregue
(`TRANSCRICAO.md` [09:09] Diego). O worker conecta no mesmo banco/`DATABASE_URL` da
API, mas instancia seu **próprio `PrismaClient`**, porque `PrismaClient` é por
processo (`TRANSCRICAO.md` [09:29][09:30] Diego, Bruno) — mesmo padrão de composição
manual de dependências usado em `src/app.ts` (`buildControllers`), sem framework de DI.

A tabela `webhook_outbox` deve ter índice em `status` (pendente/processando/falhou/
entregue) e em `created_at`, para o worker ler eficientemente os eventos pendentes mais
antigos (`TRANSCRICAO.md` [09:08] Diego), seguindo o padrão de índices já usado em
`prisma/schema.prisma` (ex.: `@@index([status])`, `@@index([createdAt])` em `Order`).

## Alternativas Consideradas

- **Trigger nativo do MySQL para notificar o worker.** Descartada: MySQL não tem
  mecanismo equivalente ao `LISTEN`/`NOTIFY` do Postgres; um trigger só executa SQL,
  não consegue notificar um processo externo. Improvisar isso (escrever em arquivo,
  bater em endpoint a partir do trigger) foi considerado "esquisito" e desnecessário,
  já que o polling de 2s já atende ao requisito de latência abaixo de 10s
  (`TRANSCRICAO.md` [09:09] Diego).
- **Worker rodando dentro do mesmo processo da API.** Implicitamente descartada
  quando Diego afirma que o worker precisa ser processo separado — reinício da API não
  pode derrubar o worker (`TRANSCRICAO.md` [09:11] Diego).

## Consequências

**Positivas**
- Ciclo de vida do worker independente da API: deploy, restart e escalonamento
  isolados.
- Latência determinística e previsível (pior caso: intervalo de polling), sem exigir
  infraestrutura de mensageria adicional.
- Implementação simples de operar e depurar (loop com `setInterval`/`setTimeout`
  recursivo, sem dependências externas novas).

**Negativas**
- Latência mínima de até 2 segundos, mesmo no melhor caso (não é "tempo real" no
  sentido estrito, mas atende à definição de <10s acordada com os clientes —
  `TRANSCRICAO.md` [09:10] Larissa, Marcos).
- Com um único worker, throughput e disponibilidade do processamento de eventos
  ficam limitados a essa instância. Escalar para múltiplos workers em paralelo
  compromete a garantia de ordering (ver limitação documentada em
  [RFC — Impacto e riscos](../RFC.md#impacto-e-riscos)) e é explicitamente colocado
  como problema futuro (`TRANSCRICAO.md` [09:13] Diego).
- Exige gerenciar mais um processo em produção (deploy, monitoramento, restart
  automático).
