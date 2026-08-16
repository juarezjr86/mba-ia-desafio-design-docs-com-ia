# Da Reunião ao Documento: Design Docs Gerados por IA

> O enunciado original do desafio foi preservado em
> [`docs-desafio/ENUNCIADO.md`](./docs-desafio/ENUNCIADO.md). Este README documenta o
> **processo de produção** da entrega, não o enunciado.

## Sobre o desafio

O desafio consiste em transformar `TRANSCRICAO.md` — a gravação literal de uma reunião
técnica de ~55 minutos sobre uma nova feature de Webhooks de Notificação de Pedidos —
e o código-fonte de um Order Management System já existente em um pacote completo de
design docs (PRD, RFC, FDD, ADRs e Tracker de rastreabilidade), usando IA como
ferramenta principal de produção. A restrição central não é técnica, é epistêmica:
nenhuma informação pode entrar nos documentos sem uma origem identificável — ou está na
transcrição, ou está no código, ou não entra. O papel do aluno é o de maestro: decidir
o que precisa ser produzido, escrever prompts dirigidos, revisar criticamente o que a
IA entrega e corrigir alucinações antes que virem "requisito documentado".

## Ferramentas de IA utilizadas

- **Claude Code (Claude Sonnet 5)** — única ferramenta usada nesta entrega. Papel:
  exploração do repositório e da transcrição, classificação estruturada do conteúdo em
  tabelas rastreáveis, redação de todos os documentos (`docs/PRD.md`, `docs/RFC.md`,
  `docs/FDD.md`, `docs/adrs/*.md`, `docs/TRACKER.md`) e deste README, sob supervisão e
  aprovação humana em cada etapa.

## Workflow adotado

O processo seguiu a ordem sugerida pelo enunciado, com um checkpoint de aprovação
humana entre a fase de exploração e a fase de produção:

1. **Setup**: clone do repositório base (`devfullcycle/mba-ia-desafio-design-docs-com-ia`).
2. **Governança do projeto**: criação de um `CLAUDE.md` específico deste repositório
   (regras de rastreabilidade, proibição de alterar código, convenções de documento),
   de `TASKS/` (backlog e tarefas por etapa) e de `licoes.md` (diário de aprendizagem).
3. **Exploração dirigida**: leitura completa de `TRANSCRICAO.md` e dos arquivos de
   código relevantes (módulos, máquina de estados, padrão de erros, middlewares,
   schema Prisma), produzindo dois mapas estruturados antes de qualquer geração de
   texto: um mapa da transcrição classificado em decisões fechadas / alternativas
   descartadas / questões em aberto / restrições / riscos (`TASKS/002`), e um mapa de
   integração com o código (`TASKS/003`).
4. **Checkpoint humano**: o plano de execução foi apresentado antes de qualquer
   documento final ser gerado, com opção explícita de revisar os mapas antes de seguir.
5. **Produção em cadeia**: ADRs → RFC → FDD → PRD → Tracker → README, cada documento
   escrito a partir dos mapas da etapa 3 (não relendo a transcrição do zero a cada
   vez), com referências cruzadas entre documentos (RFC linka ADRs, FDD linka ADRs,
   Tracker aponta para todos).

## Prompts customizados

O prompt principal desta sessão foi um prompt de orquestração de papel + processo,
adaptado a partir do material do curso, invocado via comando customizado
`/remote-control`. Trecho relevante (regra que governou toda a sessão):

```
REGRA FUNDAMENTAL — NÃO INVENTAR

Nenhum requisito, decisão, restrição, comportamento, contrato, métrica ou
característica técnica poderá ser inventado. Toda informação documentada deverá
possuir origem identificável em uma das duas fontes oficiais:
1. TRANSCRICAO.md
2. Código existente no repositório

Se uma informação não possuir origem identificável: NÃO invente; NÃO complete por
conhecimento de mercado; NÃO assuma. Classifique explicitamente como "Não
informado", "Questão em aberto", "Hipótese", "Inferência" ou "Necessita validação".
```

Esse trecho foi promovido para uma seção própria do `CLAUDE.md` deste projeto,
tornando-o uma regra persistente em vez de uma instrução de um único prompt — qualquer
sessão futura de IA neste repositório herda a mesma restrição automaticamente.

Um segundo prompt, usado internamente para dirigir a produção de cada documento
individual (adaptado do padrão do enunciado), seguia esta estrutura:

```
Para o documento <X>:
1. Use exclusivamente os itens já classificados em TASKS/002 (transcrição) e
   TASKS/003 (código) como fonte de fato.
2. Não reabra a leitura livre de TRANSCRICAO.md para "completar" o documento —
   se um dado necessário não está no mapa, ele não entra, ou entra marcado como
   hipótese.
3. Toda seção obrigatória do enunciado (<lista de seções>) precisa estar presente.
4. Ao final, verifique: nenhum item da tabela "fora de escopo" do mapa aparece
   como requisito neste documento.
```

Esse prompt dirigido — em vez de "gere um PRD a partir da transcrição" — foi o que
manteve os documentos presos aos fatos da reunião em vez de genéricos.

## Iterações e ajustes

1. **Repositório vazio no primeiro turno.** O comando inicial (`/remote-control`) foi
   disparado sobre uma pasta de projeto vazia, sem `TRANSCRICAO.md` nem código. Em vez
   de inventar contexto ou assumir uma estrutura, a sessão parou e perguntou
   explicitamente ao usuário onde estavam as fontes — o usuário indicou o repositório
   base no GitHub, que foi então clonado. Esse é o próprio princípio de "não inventar"
   aplicado antes mesmo do primeiro documento: sem fonte, sem produção.
2. **Placeholder genérico em `docs/adrs/README.md`.** O boilerplate clonado trazia um
   README de ADRs com convenção de nomenclatura diferente da exigida pelo enunciado
   (`0001-titulo.md` em vez de `ADR-NNN-titulo-em-kebab-case.md`). Foi corrigido para
   refletir a convenção real usada e transformado em índice de navegação dos 6 ADRs.
3. **Campo sem origem identificável no FDD.** Ao especificar o contrato de
   `GET /webhooks`, surgiu a pergunta implícita "a secret volta na listagem?" — a
   reunião só decide que a secret é devolvida *na criação* ([09:31] Marcos), sem
   nunca discutir listagem. Em vez de assumir silenciosamente "não, por segurança", o
   FDD marca esse ponto explicitamente como inferência de implementação (não decisão
   de reunião), e o Tracker registra a linha correspondente (`FDD-CONTRATO-08`) como
   hipótese, sem fonte de transcrição ou código — a lacuna fica visível em vez de
   escondida atrás de uma frase que parece factual.
4. **Cálculo de cobertura do Tracker.** A primeira versão da tabela do Tracker foi
   revisada linha a linha para confirmar contagem real de fontes (99 linhas, 17
   `CODIGO`, 81 `TRANSCRICAO` com timestamp) antes de declarar os critérios de ≥80% de
   cobertura e ≥70% de fonte transcrição como atendidos — não foi uma estimativa, foi
   contagem exaustiva.
5. **Revisão de profundidade sobre PRD e FDD, a pedido do usuário.** Uma segunda
   leitura crítica encontrou 4 problemas reais no FDD: (a) o exemplo de resposta de
   `GET /webhooks/:id/deliveries` omitia os campos `payload`/`response` que o próprio
   requisito ([09:34] Marcos) exige mostrar; (b) 4 dos 7 códigos `WEBHOOK_*` são
   inferência minha (seguem o padrão, mas não foram ditos em ata), e o aviso de
   inferência original estava colado no código errado — corrigido para marcar
   explicitamente quais dos 7 códigos são literais e quais são inferidos; (c) a seção
   "Fora do escopo" do FDD citava "RFC — Questões em aberto" como fonte de 5 itens,
   mas 2 deles (dashboard visual, arquivamento) nunca aparecem no RFC — corrigido para
   apontar primariamente para o PRD; (d) o código `WEBHOOK_PAYLOAD_TOO_LARGE` estava
   rotulado com status HTTP 422 mesmo ocorrendo dentro do worker assíncrono, sem
   nenhuma requisição/resposta HTTP síncrona associada — corrigido para "uso interno",
   como já era feito para `WEBHOOK_DELIVERY_TIMEOUT`. As correções propagaram também
   para duas linhas do Tracker que citavam nome de código como fonte-transcrição
   quando só o comportamento, não o nome, vinha da reunião.
6. **Data inventada nos 6 ADRs.** A linha de Status de todos os ADRs afirmava
   "decidido em reunião técnica de **2026**" — mas `TRANSCRICAO.md` nunca informa o
   ano da reunião, só "quinta-feira, 09:00". O 2026 vazou da data corrente da sessão
   (contexto do sistema) para dentro de um documento que deveria ser 100% rastreável à
   transcrição — o tipo de erro mais sutil e mais perigoso de pegar, porque "parece"
   um dado real. Corrigido nos 6 arquivos para declarar explicitamente que a data
   exata não é informada na fonte. A mesma revisão também achou uma referência cruzada
   quebrada (`ADR-002` apontava para uma inexistente "Seção 6 do RFC"; a informação
   está de fato em "Impacto e riscos") e reforçou a citação de autoria do RFC com a
   fala real da Larissa em [09:50] ("Eu vou abrir o doc de design da feature").
7. **Avaliação docente independente por um subagente Opus, sem contexto desta
   conversa.** Delegado explicitamente para funcionar como segunda opinião real (leu o
   enunciado oficial no GitHub, reavaliou o pacote do zero, recontando tudo). Achou 6
   problemas reais que as duas rodadas anteriores não pegaram — o mais grave de todos:
   a seção "Cobertura" do próprio `TRACKER.md` afirmava "100 linhas, contagem
   exaustiva verificada por busca automatizada" quando o número real de linhas de
   dados é **99** — meu `grep` de verificação anterior contava sem querer a linha de
   cabeçalho da tabela como se fosse dado. Também achou: o `README.md` citando "80
   TRANSCRICAO" quando o Tracker já tinha sido corrigido para 81 (dois documentos
   contando a mesma tabela e chegando a números diferentes); `TASKS/010` declarando
   "3 riscos" quando o PRD tem 4; o FDD deixando em aberto ("ou/ou") uma decisão de
   DLQ que o ADR-003 já tinha fechado; uma contradição entre "emite exatamente um
   evento" e "uma linha por endpoint elegível"; um `PATCH` referenciando um `GET por
   id` que não existe no documento; e 4 linhas do Tracker citando quem *levantou* um
   tema como se fosse quem *decidiu* (ex.: Marcos pergunta sobre e-mail de alerta,
   quem decide descartar é Larissa). Todos corrigidos. A lição mais importante desta
   rodada: uma afirmação sobre o próprio processo de verificação ("contei
   exaustivamente") é tão sujeita a erro quanto qualquer outra afirmação de conteúdo —
   e é mais perigosa, porque desarma quem está revisando.
8. **Quarta camada de revisão, agora com subagente Fable, instruído explicitamente a
   não confiar nas correções anteriores e reconferir tudo do zero.** Confirmou as 6
   correções da rodada Opus, exceto uma: a correção "um evento por endpoint elegível"
   tinha sido aplicada só no FDD, não no `ADR-001`, que ainda dizia "gera exatamente um
   evento" — contradição não propagada entre documentos do mesmo pacote. Achou também:
   `ADR-002` descrevendo um domínio de status da outbox (`pendente/processando/falhou/
   entregue`) que a decisão de DLQ, fechada depois na mesma reunião, tornou obsoleto
   frente ao modelo final do FDD (`pending`/`delivered`, sem status `failed`
   intermediário); uma citação `[09:36] Marcos` no PRD apontando para uma fala sem
   relação com o texto ao lado (o `[09:36]` de Marcos é sobre CRUD de webhook, não
   sobre comunicação a clientes); um caminho relativo errado no FDD (`../TRACKER.md`
   a partir de `docs/FDD.md` aponta para a raiz do repositório, não para
   `docs/TRACKER.md`); e uma linha do Tracker (limite de 64KB) atribuindo a decisão só
   a quem levantou o tema (Sofia), não a quem definiu o valor (Diego). Cada rodada de
   revisão independente encontrou resíduos que a rodada anterior — inclusive uma
   rodada de IA diferente, mais forte — não tinha propagado para todos os documentos
   do pacote. Isso é o padrão mais importante de toda a produção: correção aplicada em
   um documento não significa correção propagada a todos os documentos que compartilham
   aquela informação.
9. **Quinta camada de revisão, de volta a Opus, agora instruído a caçar
   especificamente esse padrão de propagação incompleta e a fazer uma varredura de
   consistência cruzada fato por fato.** Confirmou as 5 correções da rodada Fable e
   achou o problema mais sério de toda a produção até aqui — não um resíduo de
   correção anterior, mas uma **lacuna original**: o FDD nunca especificava o endpoint
   de rotação de secret, apesar de o PRD (`FR-11`), o RFC e o `ADR-004` exigirem
   rotação com grace period de 24h ([09:21] Sofia) como requisito fechado — e o
   próprio FDD já *pressupunha* a existência desse endpoint numa observação sobre a
   listagem ("a secret não é retornada [...] apenas na criação e **na rotação**") sem
   nunca documentá-lo. É a primeira vez que uma rodada encontra um requisito ausente
   em vez de uma contradição entre afirmações já existentes — as 4 rodadas anteriores
   focaram em coisas que os documentos diziam de forma errada ou inconsistente, e
   nenhuma tinha perguntado "o que o PRD promete que o FDD nunca chega a construir?".
   Corrigido: novo endpoint `POST /webhooks/:id/rotate-secret` no FDD (contrato,
   critério de aceite, item de escopo), nova linha no Tracker
   (`FDD-CONTRATO-09`), e números desatualizados em `TASKS/010` (contagem de
   endpoints, percentual do Tracker) e uma inconsistência textual entre README e
   `licoes.md` sobre qual rodada era a "terceira" vs. "quarta" camada de revisão.
10. **Sexta camada de revisão, em Sonnet, focada em verificar se a correção da
    rodada anterior (endpoint de rotação de secret) foi propagada por completo.**
    O endpoint em si estava correto, mas 2 lugares que resumem a superfície de API
    não tinham sido atualizados: a "Superfície de API proposta" do RFC (lista os
    grupos de endpoint em prosa, mas não citava a rotação) e a matriz de erros do FDD
    (a linha `WEBHOOK_NOT_FOUND` não mencionava o novo endpoint entre os gatilhos).
    Corrigidos. Esta rodada também foi a primeira a recontar todos os números
    auto-relatados do pacote (Tracker, riscos, endpoints, ADRs) contra a contagem real
    dos arquivos — sinal de que a precisão numérica estava convergindo, mesmo com um
    resíduo de propagação ainda aparecendo.
11. **Auditoria de estado antes da entrega final, com recontagem independente por
    `grep` de cada número auto-relatado.** As contagens de RFs (12), RNFs (7),
    endpoints (7), códigos `WEBHOOK_` (7), ADRs (6) e fontes do Tracker (17 `CODIGO`,
    83 `TRANSCRICAO`) conferiram — com uma ressalva que só a iteração 12 pegaria: a
    contagem de endpoints saiu errada (9) na primeira redação deste parágrafo, embora
    o FDD sempre tenha tido 7. O achado desta rodada foi uma lacuna de cobertura em
    sentido inverso ao da iteração 9: lá o PRD prometia algo que o FDD não construía; aqui o
    PRD listava um risco real — "worker fica fora do ar e para de processar a outbox"
    ([09:11] Diego) — que **nenhuma linha do Tracker cobria** do lado do PRD, embora o
    mesmo fato aparecesse rastreado a partir do FDD (`FDD-RISK-01`). O item existia e
    tinha origem legítima; o que faltava era o vínculo de rastreabilidade. Corrigido
    com a linha `PRD-RISK-04` e a recontagem da seção "Cobertura" do Tracker (100 →
    101 linhas). Lição embutida: verificar cobertura do Tracker por *documento de
    origem*, não só pelo total agregado — um percentual global saudável esconde
    seções inteiras sem rastreabilidade.
12. **Avaliação final simulada em Fable, com a IA no papel do professor que escreveu o
    enunciado, avaliando o pacote contra os próprios critérios de aceite.** Parecer:
    aprovado, sem nenhuma falha grave — os critérios de aceite atendidos, 12
    afirmações factuais reconferidas contra a transcrição e 17/17 caminhos de código
    confirmados como existentes. A rodada encontrou duas correções, ambas de
    metadiscurso e não de conteúdo: (a) a contagem de endpoints na iteração 11 dizia 9
    onde o FDD tem 7 — o erro nasceu de contar os IDs `FDD-CONTRATO-01..09` do Tracker
    como se todos fossem endpoints, quando dois deles (`-06` payload de entrega, `-07`
    headers) são contratos de mensagem; (b) a frase de abertura dos contratos do FDD
    dizia que "todos os endpoints ficam sob `/api/v1/webhooks`", enquanto o replay
    administrativo fica sob `/api/v1/admin/webhooks` — o próprio FDD já registrava os
    dois routers corretamente na seção de integração. Lição embutida, e a mais dura do
    projeto: o número que descreve o *trabalho* recebeu, durante toda a produção, menos
    escrutínio que o número que descreve o *sistema* — e o erro sobreviveu justamente
    dentro do parágrafo que celebrava a recontagem por `grep`. Afirmação sobre o
    próprio rigor desarma o revisor, inclusive o que a escreveu.

## Como navegar a entrega

Ordem de leitura recomendada:

1. [`docs/PRD.md`](./docs/PRD.md) — por que a feature existe e o que ela precisa fazer.
2. [`docs/RFC.md`](./docs/RFC.md) — proposta técnica de alto nível, alternativas e
   questões em aberto.
3. [`docs/adrs/`](./docs/adrs/README.md) — as 6 decisões arquiteturais isoladas que
   sustentam o RFC.
4. [`docs/FDD.md`](./docs/FDD.md) — especificação de implementação: fluxos, contratos
   HTTP, matriz de erros, integração com o código existente.
5. [`docs/TRACKER.md`](./docs/TRACKER.md) — para verificar a origem de qualquer
   afirmação nos documentos acima.
6. [`TASKS/`](./TASKS/000_BACKLOG.md) — histórico do processo, tarefa por tarefa.
7. [`licoes.md`](./licoes.md) — diário de aprendizagem da sessão.
8. [`CLAUDE.md`](./CLAUDE.md) — regras que governaram esta sessão de IA neste
   repositório.

O código-fonte da aplicação (`src/`, `prisma/`, `tests/`) não foi alterado nesta
entrega — serve apenas como referência, conforme exigido pelo enunciado.
