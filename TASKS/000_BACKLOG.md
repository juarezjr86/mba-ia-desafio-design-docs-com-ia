# TASK 000 — Backlog Geral

## Objetivo

Servir como índice mestre de todas as tarefas do projeto e como tabela de
acompanhamento dos entregáveis.

## Contexto

Desafio acadêmico: transformar `TRANSCRICAO.md` + código existente do OMS em pacote de
design docs (PRD, RFC, FDD, ADRs, Tracker, README). Ver `CLAUDE.md` para regras
completas e `docs-desafio/ENUNCIADO.md` para o enunciado original.

## Entregáveis

| Entregável | Estado |
| --- | --- |
| `docs/PRD.md` | 🟢 |
| `docs/RFC.md` | 🟢 |
| `docs/FDD.md` | 🟢 |
| `docs/TRACKER.md` | 🟢 |
| `docs/adrs/` (6 ADRs) | 🟢 |
| `README.md` (processo) | 🟢 |
| `licoes.md` | 🟢 |
| `CLAUDE.md` | 🟢 |
| `TASKS/` | 🟢 |
| Auditoria final | 🟢 |
| Critérios de aceite | 🟢 |

## Status

Concluído. Todos os entregáveis produzidos e auditados contra a checklist do
enunciado (ver `TASKS/010_REVISAO_FINAL.md`).

## Dependências

Nenhuma.

## Itens pendentes

Nenhum item de produção. Pendências de responsabilidade do aluno, fora do escopo desta
sessão: revisar o pacote com olhar crítico próprio, e decidir sobre `git add`/`commit`/
`push` para o fork público no GitHub exigido pelo enunciado.

## Itens concluídos

- Clone do repositório base do desafio.
- Exploração de código e transcrição (`TASKS/001`–`003`).
- 6 ADRs (`docs/adrs/`).
- RFC (`docs/RFC.md`), linkando os 6 ADRs.
- FDD (`docs/FDD.md`), com seção "Integração com o sistema existente" (6 caminhos
  reais) e matriz de erros `WEBHOOK_*`.
- PRD (`docs/PRD.md`), com 12 requisitos funcionais e métrica quantitativa de latência.
- Tracker (`docs/TRACKER.md`), 99 linhas, ~81% fonte TRANSCRICAO, 17 fonte CODIGO.
- README do processo, com 2 prompts customizados e 4 iterações documentadas.
- Auditoria final (`TASKS/010_REVISAO_FINAL.md`) — todos os critérios do enunciado
  confirmados via contagem automatizada, não estimativa.
- Confirmado, via `git status`, que nenhum arquivo de `src/`, `prisma/`, `tests/` ou
  configuração foi alterado.

## Riscos

Nenhum risco em aberto na produção. Risco residual fora do controle desta sessão:
revisão crítica humana do aluno ainda não realizada — a entrega é uma primeira versão
completa e auditada automaticamente contra os critérios objetivos do enunciado, não
uma revisão editorial humana.

## Próximos passos

Aluno revisa o pacote (`docs/`, `README.md`) e decide sobre push ao fork público.

## Observações

Nenhuma.
