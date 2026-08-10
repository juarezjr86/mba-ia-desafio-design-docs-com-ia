# CLAUDE.md — Projeto Acadêmico: Da Reunião ao Documento (Design Docs com IA)

## Contexto acadêmico

Este repositório é a entrega de um desafio do MBA em Desenvolvimento de Sistemas com
Ênfase em IA. **Não é um projeto de implementação.** O código em `src/`, `prisma/` e
`tests/` é um Order Management System (OMS) já funcional, fornecido como **contexto e
referência**, não como alvo de desenvolvimento.

O objetivo do desafio é transformar `TRANSCRICAO.md` (reunião técnica sobre uma nova
feature de Webhooks de Notificação de Pedidos) + o código existente em um pacote de
design docs de nível profissional: PRD, RFC, FDD, 5–8 ADRs, Tracker de rastreabilidade
e um README documentando o processo de produção.

A IA (Claude Code) atua como ferramenta principal de produção; o aluno (usuário) atua
como maestro — define o que precisa, escreve/ajusta prompts, revisa criticamente,
corrige alucinações. Este CLAUDE.md governa como a IA deve se comportar neste
repositório especificamente. Instruções globais do usuário (`~/.claude/CLAUDE.md`) que
tratam de ADVPL/TLPP/Protheus **não se aplicam aqui** — este projeto é Node.js/TypeScript
e puramente documental.

## Regra fundamental: não inventar

Toda informação em qualquer documento entregue (PRD, RFC, FDD, ADR, Tracker, README)
precisa ter origem rastreável em uma das duas fontes oficiais:

1. `TRANSCRICAO.md` — citar como `[hh:mm] Nome`
2. Código-fonte existente — citar como caminho real do arquivo

Se uma informação não tem origem identificável: **não inventar**. Classificar
explicitamente como "Não informado", "Questão em aberto", "Hipótese", "Inferência" ou
"Necessita validação", e registrar a origem sempre que possível.

Antes de mencionar qualquer arquivo de código na documentação, confirmar que ele existe
(Read/Glob/Grep) — nunca citar caminho por suposição.

## Proibição absoluta de alteração do código

**Nunca editar** `src/`, `prisma/`, `tests/`, `package.json`, `package-lock.json`,
`tsconfig.*`, `vitest.*`, `.eslintrc.*`, `docker-compose.yml`, `.env.example` ou
qualquer arquivo de configuração da aplicação. O código é somente leitura durante todo
o projeto. A entrega é 100% documental, dentro de `docs/`, `TASKS/`, `licoes.md`,
`README.md` e este `CLAUDE.md`.

## Fontes oficiais

- `TRANSCRICAO.md` — reunião de ~55 min entre Larissa (Tech Lead), Marcos (PM), Bruno
  (Eng. Pleno, time de Pedidos), Diego (Eng. Sênior, time de Plataforma) e Sofia
  (Eng. de Segurança). Contém decisões fechadas, requisitos, alternativas descartadas,
  itens adiados e questões em aberto.
- Código em `src/`, `prisma/schema.prisma`, `tests/` — padrões reais a reutilizar
  (módulos, `AppError`, `requireRole`, logger Pino, error middleware, máquina de
  estados de `OrderStatus`, transação de `changeStatus`).
- Enunciado original do desafio (preservado em `docs-desafio/ENUNCIADO.md` ou na seção
  "Sobre o desafio" do novo README) — define os critérios de aceite obrigatórios.

## Estrutura dos documentos e regra de altura

Os documentos não se repetem — cada um responde uma pergunta diferente:

| Documento | Pergunta | Altura |
| --- | --- | --- |
| PRD | Por que e o quê? | Produto/negócio |
| RFC | Como propomos resolver, e o que está em aberto? | Arquitetura (conciso, 2–4 páginas) |
| ADR | Por que decidimos exatamente assim? | Decisão pontual |
| FDD | Como construir, em detalhe? | Implementação |
| Tracker | De onde veio cada informação? | Transversal |

Conteúdo duplicado entre documentos (ex: RFC repetindo payloads JSON do FDD) é sinal de
que algo está na altura errada — deve ser cortado do documento de altura mais alta e
deixado só no mais detalhado.

## Regras para ADRs

- Local: `docs/adrs/`, nome `ADR-NNN-titulo-em-kebab-case.md`, numeração sequencial
  3 dígitos.
- Seções mínimas (formato MADR): Status, Contexto, Decisão, Alternativas Consideradas
  (≥1 alternativa real discutida ou plausível), Consequências (positivas e negativas,
  trade-off explícito).
- Entre 5 e 8 ADRs cobrindo ao menos 5 das 6 decisões principais da reunião: Outbox no
  MySQL, retry com backoff e DLQ, HMAC-SHA256 com secret por endpoint, at-least-once
  com `X-Event-Id`, worker em processo separado com polling, reuso dos padrões
  existentes.
- Pelo menos 1 ADR deve referenciar explicitamente arquivo/módulo/classe real do
  código-base.

## Regras para o Tracker

- Tabela única em `docs/TRACKER.md`, colunas: ID | Documento | Tipo | Conteúdo (resumo)
  | Fonte | Localização.
- Fonte só pode ser `TRANSCRICAO` (localização = `[hh:mm] Nome`) ou `CODIGO`
  (localização = caminho real do arquivo).
- Cobertura ≥80% dos itens identificáveis nos documentos; ≥70% das linhas com Fonte =
  TRANSCRICAO; ≥5 linhas com Fonte = CODIGO.
- Se não for possível preencher "Localização" para um item, é sinal de alucinação —
  remover ou corrigir o item na fonte, não forçar uma linha no tracker.

## Processo de revisão / estratégia de validação

Ciclo obrigatório por documento: ANALISAR → GERAR → REVISAR → IDENTIFICAR PROBLEMAS →
CORRIGIR PROMPT → REFINAR → VALIDAR → GERAR NOVAMENTE. Esperado de 3 a 5 ciclos no
total do projeto. Documentos genéricos ou que "cheiram" a transcrição colada são sinal
de prompt fraco — refinar, não aceitar de primeira.

Antes de declarar qualquer documento concluído, checar:

1. Toda seção obrigatória do enunciado está presente.
2. Todo requisito/decisão tem origem rastreável (Tracker).
3. Nenhum item explicitamente descartado ou adiado na reunião aparece como requisito.
4. Nenhum arquivo de código citado é inexistente.
5. Documento não duplica a altura de outro documento do pacote.

## Uso do Superpowers

Usar as skills do Superpowers durante exploração, decomposição, planejamento e revisão
crítica — não pular direto para geração de texto. Em particular:
`systematic-debugging`-style de investigação para achar inconsistências,
`verification-before-completion` antes de marcar qualquer entregável como concluído.

## Comportamento esperado da IA (modo professor)

Para cada etapa relevante, explicar em duas camadas: (1) explicação simples do que
está sendo feito e por quê; (2) camada técnica — conceito, aplicação no projeto,
trade-offs, alternativas. Registrar aprendizado em `licoes.md` (nunca apagar lições
anteriores). Não produzir documento final sem antes mapear e apresentar um plano ao
aluno.

## Convenções Markdown

- Markdown puro, sem HTML embutido salvo quando estritamente necessário.
- Tabelas para todo conteúdo tabular (requisitos, matriz de erros, tracker).
- Links relativos entre documentos do pacote (ex: RFC → `./adrs/ADR-001-....md`).
- Sem emojis no conteúdo técnico (uso permitido apenas nos estados da tabela de
  entregáveis: ⬜ 🟡 🟢 🔴 ⚠️).

## Regras contra alucinação (resumo operacional)

- Nunca completar lacuna de informação com "conhecimento de mercado" apresentado como
  fato do projeto — se for inferência de mercado (ex: "Stripe faz assim"), só entra se
  estiver **literalmente dita na transcrição** (é o caso: Diego cita Stripe/GitHub em
  [09:25]).
- Números (5 tentativas, 2s de polling, 64KB, 24h de grace period, 10s de timeout)
  devem bater exatamente com o que foi dito na reunião — não arredondar, não "melhorar".
  Se dois números conflitarem, é uma correção de prompt anterior, não a IA definida.
- Itens descartados (email de alerta, dashboard visual, rate limiting de saída,
  garantia de ordering global) **nunca** viram requisito — no máximo entram em "Fora de
  escopo" ou "Questões em aberto".
