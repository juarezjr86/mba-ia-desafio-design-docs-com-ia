# Lições — Diário de Aprendizagem

Este arquivo registra o aprendizado ao longo da produção do pacote de design docs.
Nunca é apagado; apenas recebe novas lições.

---

# Lição 01 — Da transcrição bruta ao mapa estruturado

## Objetivo

Entender por que a exploração (código + transcrição) precisa vir *antes* de qualquer
prompt de geração de documento, e como transformar uma reunião de 55 minutos em um
mapa que uma IA (ou uma pessoa) consegue usar sem alucinar.

## O que analisamos

- O enunciado do desafio (`docs-desafio/ENUNCIADO.md`), que define exatamente quais
  documentos produzir e com quais critérios de aceite objetivos (contagens mínimas,
  seções obrigatórias).
- O código-fonte do OMS existente: estrutura modular, máquina de estados de pedido,
  padrão de erros, middleware de auth, logger, schema Prisma.
- `TRANSCRICAO.md` completa, linha a linha, com timestamp e falante.

## O que foi produzido

- `CLAUDE.md` do projeto (regras específicas deste repositório, distintas das regras
  ADVPL do CLAUDE.md global do usuário).
- `TASKS/000` a `TASKS/003`: backlog, mapa de exploração, mapa da transcrição
  classificado (decisões fechadas / alternativas descartadas / questões em aberto /
  restrições / riscos) e mapa de integração com o código.

## Por que fizemos dessa forma

Um LLM instruído a "gerar um PRD a partir dessa transcrição" tende a resumir de forma
genérica e a preencher lacunas com conhecimento de mercado — silenciosamente. A defesa
contra isso não é pedir "com cuidado" no prompt seguinte: é produzir, antes, um mapa
onde cada afirmação já está presa a uma fonte (timestamp ou caminho de arquivo). Depois
disso, escrever o PRD/RFC/FDD é essencialmente *transcrever do mapa*, não *interpretar a
transcrição de novo* — o que reduz a superfície de alucinação.

## Conceitos aprendidos

- **Requisito vs. menção**: nem tudo que é dito na reunião é requisito. Itens
  descartados (síncrono, Redis, retry indefinido, 3 tentativas, truncar payload,
  exactly-once) e itens adiados (e-mail de alerta, dashboard, rate limiting de saída)
  são tão importantes de registrar quanto o que foi decidido — porque a armadilha mais
  comum é a IA "recuperar" uma ideia descartada e apresentá-la como requisito.
- **Altura de documento**: PRD, RFC, ADR, FDD e Tracker respondem perguntas diferentes.
  Decidir a altura de cada informação *antes* de escrever evita duplicação.
- **Rastreabilidade como mecanismo de verificação, não de burocracia**: o Tracker não é
  um "documento a mais" — é o teste de hipótese "essa frase do PRD é real?". Se não dá
  para apontar `[hh:mm] Nome` ou um caminho de arquivo, a frase provavelmente foi
  inventada.

## Tecnologias e ferramentas envolvidas

Claude Code (exploração de repositório, leitura de arquivos, clonagem via `git`),
Markdown como formato de saída, GitHub (repositório base do desafio,
`devfullcycle/mba-ia-desafio-design-docs-com-ia`).

## Conceitos de Engenharia de Software

- Padrão **Outbox** (consistência entre mudança de estado transacional e emissão de
  evento assíncrono) — decidido em [09:06]–[09:08] da reunião, com trade-off explícito
  contra síncrono e contra fila externa.
- Máquina de estados finita aplicada a ciclo de vida de pedido (`order.status.ts`).
- Separação processo API / processo worker como forma de isolar falha e ciclo de vida.

## O que poderia dar errado

- A IA poderia ter citado um arquivo de código plausível mas inexistente (ex:
  `src/modules/webhooks/webhook.service.ts` antes mesmo de a feature existir) como se
  já existisse. Mitigado: toda citação de arquivo em TASKS/001 e TASKS/003 foi
  confirmada via leitura direta; caminhos de arquivos *futuros* (que a feature vai
  criar) são citados apenas nas seções de proposta (RFC/FDD/ADR), nunca como "código
  existente".
- Números da reunião (2s, 5 tentativas, 64KB, 24h, 10s) poderiam ser "arredondados" ou
  confundidos entre si num resumo solto. Mitigado registrando-os em tabela com
  timestamp exato antes de qualquer prosa.

## Como a IA foi utilizada

Claude Code leu integralmente `TRANSCRICAO.md` e os arquivos de código relevantes via
tool `Read`, sem resumir por amostragem, e organizou o conteúdo em tabelas
classificadas antes de qualquer produção de PRD/RFC/FDD/ADR.

## O que o aluno deve compreender

Que o valor do processo está em separar **coleta e classificação de fatos** (esta
etapa) de **redação do documento final** (próxima etapa). Pular direto para "gera o
PRD" é o erro mais comum e mais caro de corrigir depois.

## Resumo

Antes de escrever qualquer design doc, mapeamos código e transcrição em tabelas
rastreáveis. Esse mapa (TASKS/001–003) é a fonte de verdade para tudo que vem depois.

## Próximos passos

Apresentar o plano de execução ao aluno para validação, depois iniciar os ADRs
(TASKS/004), seguindo a ordem: ADRs → RFC → FDD → PRD → Tracker → README.

---

# Lição 02 — Produzir na ordem certa e provar a cobertura, não estimá-la

## Objetivo

Entender por que a ordem ADRs → RFC → FDD → PRD → Tracker → README não é arbitrária, e
por que "o Tracker cobre 80%" só vale alguma coisa se for contado, não sentido.

## O que analisamos

Os mapas produzidos na Lição 01 (`TASKS/002`, `TASKS/003`), usados como única fonte de
fato para os 6 documentos finais — nenhuma releitura livre da transcrição durante a
redação.

## O que foi produzido

`docs/adrs/ADR-001` a `ADR-006`, `docs/RFC.md`, `docs/FDD.md`, `docs/PRD.md`,
`docs/TRACKER.md` (99 linhas) e `README.md` do processo.

## Por que fizemos dessa forma

Cada documento seguinte reaproveitou o anterior em vez de voltar à transcrição: o RFC
referenciou os ADRs em vez de reexplicar cada decisão; o FDD expandiu o RFC em vez de
duplicá-lo; o PRD consolidou RFC+FDD+ADRs em linguagem de produto. Essa é a aplicação
prática da regra de "altura" dos documentos do `CLAUDE.md`: cada documento é escrito
*sobre* o anterior, não *ao lado* dele. O Tracker, por ser transversal, só faz sentido
por último — é o documento que teria que ser refeito inteiro se qualquer um dos outros
mudasse de conteúdo.

## Conceitos aprendidos

- **Verificação automatizada > leitura visual para critérios objetivos**: em vez de
  "acho que tem uns 12 requisitos", rodei `grep -c` para contar de fato. O mesmo para
  contagem de linhas do Tracker por fonte — a primeira estimativa (15 CODIGO / 83
  TRANSCRICAO) estava errada; a contagem real deu 17/80. Sem o `grep`, o erro teria ido
  para a entrega.
- **Marcar a lacuna é mais honesto que preencher com uma suposição plausível**: o campo
  "a secret volta na listagem de webhooks?" não foi decidido na reunião. A tentação é
  responder "não, por segurança" como se fosse óbvio e mover em frente. Em vez disso, o
  FDD registra isso como inferência de implementação, e o Tracker tem uma linha
  explicitamente sem fonte — porque de fato não tem.

## Tecnologias e ferramentas envolvidas

Claude Code (redação dos documentos, `grep`/`ls` via Bash para verificação
quantitativa dos critérios de aceite antes de declarar conclusão).

## Conceitos de Engenharia de Software

- **MADR** (Markdown Architectural Decision Records) como formato de ADR.
- Diferenciação de altura entre documento de proposta (RFC), documento de decisão
  (ADR) e documento de implementação (FDD) — mapeia diretamente à prática real de
  design docs em empresas de engenharia madura.
- Rastreabilidade como métrica auditável (percentual por fonte), não como
  característica qualitativa vaga.

## O que poderia dar errado

- Duplicar conteúdo entre RFC e FDD (ex.: repetir o payload JSON completo no RFC).
  Evitado revisando cada seção do RFC contra a régua "isso é proposta de alto nível ou
  detalhe de implementação?" antes de fechar o documento.
- Requisito fora de escopo vazando para dentro do PRD como requisito funcional (ex.:
  e-mail de alerta). Evitado cruzando cada requisito funcional do PRD contra a lista de
  itens explicitamente descartados/adiados do `TASKS/002` antes de finalizar.

## Como a IA foi utilizada

Claude Code redigiu os 6 documentos e depois auditou as próprias contagens via
comandos de shell (`grep -c`, `ls`) em vez de confiar em contagem visual, corrigindo o
texto de cobertura do Tracker quando a contagem real divergiu da estimativa inicial.

## O que o aluno deve compreender

Que "critério de aceite atendido" precisa ser verificável por um terceiro sem
confiar na palavra de quem escreveu — por isso a auditoria final (`TASKS/010`) refaz a
contagem de cada critério numérico do enunciado, em vez de apenas reafirmar que "está
tudo certo".

## Resumo

Produzimos os 6 documentos na ordem que respeita a dependência de conteúdo entre eles,
e fechamos com uma auditoria que recontou cada critério numérico do enunciado em vez
de assumi-lo como atendido.

## Próximos passos

Revisão crítica de profundidade a pedido do aluno (ver Lição 03).

---

# Lição 03 — A alucinação mais perigosa é a que parece um dado real

## Objetivo

Entender uma categoria de erro diferente da Lição 01: não é o modelo inventando
conteúdo por preguiça de checar a fonte, é o **contexto da sessão vazando para dentro
do documento** como se fosse fato da fonte.

## O que analisamos

Segunda leitura crítica, a pedido do aluno, de todos os 6 documentos (PRD, FDD, RFC,
6 ADRs, Tracker), linha a linha, cruzando cada afirmação contra `TRANSCRICAO.md` e o
código — não apenas confiando na primeira geração.

## O que foi produzido

6 correções no FDD (rodada anterior) e, nesta rodada, 3 correções adicionais: remoção
de um ano fabricado ("2026") da linha de Status dos 6 ADRs, correção de uma referência
cruzada quebrada (ADR-002 apontava para uma "Seção 6 do RFC" inexistente) e reforço da
citação de autoria do RFC.

## Por que fizemos dessa forma

O caso mais interessante desta rodada foi o ano "2026" nos ADRs. Não é uma invenção de
conteúdo de negócio — é a data corrente da sessão (fornecida ao modelo como contexto
de sistema) vazando para dentro de uma linha de "Status", escrita em tom afirmativo
("decidido em reunião técnica de 2026"), como se fosse um dado extraído da
transcrição. `TRANSCRICAO.md` nunca menciona o ano — só "quinta-feira, 09:00". Esse
tipo de erro é mais perigoso que uma alucinação óbvia porque tem a forma exata de um
fato correto: uma data, num campo de metadado, num documento que parece 100%
rastreável. Ninguém revisando "por cima" pegaria isso — só cruzando contra a fonte
literal, campo a campo.

## Conceitos aprendidos

- **Duas categorias distintas de alucinação**: (1) inventar conteúdo por não achar a
  informação na fonte (ex.: um campo de payload que não foi discutido — Lição 01/02);
  (2) vazamento de contexto da sessão (data, ambiente, convenções do modelo) para
  dentro do documento, sem qualquer intenção de "inventar" — mais traiçoeira porque
  não parece uma lacuna, parece um dado preenchido corretamente.
- **Referência cruzada é código, não decoração**: "ver Seção 6 do RFC" é uma afirmação
  verificável (existe? aponta pro lugar certo?) tanto quanto "o método changeStatus
  faz X". Deve ser auditada com o mesmo rigor.

## Tecnologias e ferramentas envolvidas

Claude Code, leitura cruzada de todos os documentos do pacote na mesma sessão de
revisão (não documento por documento isoladamente, o que teria escondido a referência
cruzada quebrada entre ADR-002 e RFC).

## Conceitos de Engenharia de Software

Analogia direta com vazamento de estado de teste (ex.: um relógio mockado que vaza pra
produção) ou com data hardcoded pega em code review — o princípio de isolamento entre
"contexto de execução" e "dado de domínio" vale tanto para código quanto para
documentação gerada por IA.

## O que poderia dar errado

Se a auditoria de profundidade tivesse revisado cada documento isoladamente em vez de
ler o pacote inteiro na mesma janela de contexto, a referência cruzada quebrada
(ADR-002 → RFC) provavelmente não teria sido pega, porque exige comparar dois
documentos ao mesmo tempo, não só validar um contra a transcrição.

## Como a IA foi utilizada

Claude Code releu os 6 ADRs, o RFC e o Tracker na mesma sessão, comparando cada
afirmação contra `TRANSCRICAO.md` e contra os outros documentos do pacote (não só
contra a fonte primária), reportou os achados de forma estruturada antes de aplicar
qualquer correção, e propagou cada correção para o Tracker e o README.

## O que o aluno deve compreender

Que revisar documentação gerada por IA não é só "isso parece plausível?" — é
"cada campo específico bate com a fonte, incluindo metadados que parecem triviais
como data e número de seção?". A plausibilidade de um dado não é evidência da sua
origem.

## Resumo

Uma segunda rodada de revisão, cobrindo os documentos que a primeira rodada não tinha
auditado a fundo, encontrou um vazamento sutil de contexto de sessão (data
"2026" apresentada como fato da reunião) e uma referência cruzada quebrada entre
documentos — ambos corrigidos e propagados ao Tracker e ao README.

## Próximos passos

Entrega revisada pelo aluno, commitada e enviada ao fork público (ver Lição 04).

---

# Lição 04 — Um segundo avaliador independente vale mais do que uma terceira releitura

## Objetivo

Entender por que, depois de duas rodadas de autorrevisão, a forma mais eficaz de achar
o que restava de errado não foi "revisar de novo com mais cuidado" — foi delegar a
avaliação para um agente novo, sem o contexto (e o viés) de quem produziu e já
revisou os documentos duas vezes.

## O que analisamos

Pedido explícito do aluno: gerar um prompt e delegar a um subagente rodando em um
modelo mais forte (Opus) para assumir o papel de professor, ler o enunciado oficial
direto do repositório do desafio no GitHub, e reavaliar a entrega do zero — sem
herdar nada desta conversa.

## O que foi produzido

Um relatório de avaliação docente completo (resumo executivo, tabela por entregável,
lista do que está coerente, lista do que diverge, veredito), e, a partir dele, 10
correções aplicadas: 3 números de autoauditoria errados (`docs/TRACKER.md` dizia "100
linhas" quando são 99; `README.md` dizia "80 TRANSCRICAO" quando o Tracker já tinha
81; `TASKS/010` dizia "3 riscos" quando o PRD tem 4), uma ambiguidade de DLQ no FDD que
contradizia uma decisão já fechada no ADR-003, uma contradição interna sobre "um
evento" vs. "um evento por endpoint elegível", um `PATCH` no FDD que referenciava um
endpoint `GET por id` inexistente no próprio documento, dois resíduos de texto em
inglês ("also", "essencially") sobrevivendo a seis iterações de revisão, e 4 linhas do
Tracker atribuindo uma decisão a quem só levantou o tema, não a quem decidiu.

## Por que fizemos dessa forma

Nas duas rodadas anteriores (Lições 02 e 03), eu estava revisando meu próprio
trabalho, com a memória de tudo que já tinha checado — um viés de confirmação real:
depois de verificar um número duas vezes e achá-lo certo, a terceira verificação tende
a ser mais rápida e mais confiante, não mais rigorosa. Meu próprio `grep` de
verificação da Lição 02 tinha um bug (contava a linha de cabeçalho da tabela do
Tracker como se fosse uma linha de dado, inflando a contagem em +1) — e eu *confiei*
nesse número errado duas vezes seguidas porque ele "batia" com o texto que eu mesmo
tinha escrito. Um agente novo, com um prompt que pedia explicitamente para recontar
tudo manualmente e não confiar em nenhuma alegação de contagem já feita dentro dos
documentos, não carrega esse viés — ele não tem motivo para confiar em um número só
porque já apareceu em três lugares.

## Conceitos aprendidos

- **Revisar o próprio trabalho tem um teto.** Depois de duas rodadas, os erros que
  sobram tendem a ser exatamente os que a pessoa (ou IA) que escreveu o documento tem
  mais dificuldade de ver — porque ela já decidiu, nas rodadas anteriores, que aquele
  trecho estava certo.
- **Delegar para um agente sem contexto não é redundância, é uma técnica de
  verificação.** O valor específico de um subagente "fresh" (sem herdar a conversa) é
  não herdar também os erros de raciocínio já cometidos — inclusive erros sobre "o que
  já foi checado".
- **O prompt de avaliação precisa proibir explicitamente confiar em números
  pré-calculados.** A instrução "reconte você mesmo, não confie em nenhuma contagem já
  feita dentro dos próprios documentos" foi o que efetivamente pegou o bug do `grep`
  — sem essa instrução, um avaliador só leria "100 linhas, contagem exaustiva
  verificada" e teria aceitado a alegação pelo tom de confiança do texto, não pelo
  número em si.

## Tecnologias e ferramentas envolvidas

Claude Code (Agent tool, subagente `general-purpose` em Opus, sem herdar contexto da
conversa principal — só o prompt escrito explicitamente para essa tarefa).

## Conceitos de Engenharia de Software

Equivalente direto a **code review por um segundo revisor humano** depois que o autor
já se autorrevisou: a literatura de qualidade de software mostra reiteradamente que
autorrevisão tem taxa de detecção de defeitos menor que revisão por terceiros,
justamente pelo mesmo motivo — familiaridade com o próprio trabalho reduz a atenção a
detalhes já "aceitos" mentalmente.

## O que poderia dar errado

Se o prompt de delegação não tivesse mandado o subagente ler o enunciado oficial
*direto do GitHub* (em vez de confiar na cópia local `docs-desafio/ENUNCIADO.md`, que
eu mesmo salvei), e se essa cópia local tivesse sido alterada por engano em algum
momento, a avaliação teria herdado esse erro silenciosamente. Por isso o prompt pediu
a fonte remota como primária e a local só como fallback.

## Como a IA foi utilizada

Um subagente Opus, delegado via `Agent` tool com um prompt de ~700 palavras
detalhando papel, fontes de verdade, o que avaliar item a item, e o formato de saída
exigido, rodou de forma assíncrona e devolveu um relatório estruturado. Cada achado
foi reverificado manualmente (com `grep`/leitura direta) antes de qualquer correção
ser aplicada — inclusive porque um dos achados do próprio relatório (a contagem "99,
não 100") contradizia uma verificação minha anterior, e era preciso decidir quem
estava certo antes de agir.

## O que o aluno deve compreender

Que pedir uma segunda opinião independente — de um agente sem o contexto e sem o viés
de quem já revisou — é uma técnica de verificação legítima e mais forte que pedir para
o mesmo agente "revisar de novo". E que mesmo o relatório de um avaliador externo
merece ser conferido, não aceito de olhos fechados: um dos achados do subagente foi
verificado contra o arquivo real antes de eu confiar nele, porque "um agente disse que
está errado" não é mais confiável por si só do que "um agente disse que está certo".

## Resumo

Delegar a avaliação final para um subagente Opus sem contexto da conversa encontrou 6
problemas reais que duas rodadas de autorrevisão não pegaram — a maioria envolvendo
números que o próprio pacote usava para provar sua própria integridade. Todos
corrigidos e propagados entre os documentos.

## Adendo — a mesma técnica, aplicada de novo, achou o que a primeira rodada deixou passar

Depois de commitar e publicar as correções da avaliação Opus, uma **quarta camada** —
outro subagente independente, desta vez em Fable, instruído explicitamente a não
confiar nas correções anteriores e reconferir tudo — encontrou o padrão mais
importante de toda a produção: a correção "um evento por endpoint elegível" tinha sido
aplicada no FDD, mas não no `ADR-001`, que ainda afirmava "gera exatamente um evento".
Mesmo problema, documento diferente, não propagado. O mesmo aconteceu com o domínio de
status da outbox: o `ADR-002` ainda descrevia `pendente/processando/falhou/entregue`
enquanto o FDD (já corrigido) tinha fixado o modelo final em `pending`/`delivered`
sem status intermediário.

Isso generaliza a lição: **corrigir um documento não corrige o pacote.** Numa
documentação com múltiplos arquivos que se referenciam e se repetem em alturas
diferentes (a mesma decisão aparece no ADR, no RFC e no FDD, cada um com seu nível de
detalhe), toda correção precisa ser buscada em todos os lugares onde a informação
original também apareceu — e isso não escala bem para revisão manual feita pela mesma
pessoa/IA que já decidiu, em rodadas anteriores, que "já corrigi isso". Cada rodada de
revisão independente sucessiva (autor → autorrevisão → Opus → Fable) encontrou algo
que a anterior não viu, incluindo resíduos das correções da rodada logo antes dela.
Isso não é sinal de processo ruim — é o oposto: é o processo funcionando como
desenhado. A pergunta não é "quando vamos parar de achar problemas", é "quando o custo
de mais uma rodada deixa de valer a pena frente ao que resta de risco real" — e depois
de 4 camadas, sem nenhum item novo tocando um critério de aceite obrigatório, esse
ponto foi atingido.

## Segundo adendo — a 5ª camada achou o que as 4 anteriores não sabiam procurar

Uma quinta rodada (Opus de novo, desta vez instruído explicitamente a caçar
propagação incompleta entre documentos) confirmou as 5 correções da rodada Fable e
achou algo de natureza diferente de tudo até então: não uma contradição entre duas
afirmações existentes, mas um **requisito inteiro ausente** do documento que deveria
implementá-lo. O PRD, o RFC e o `ADR-004` prometem rotação de secret com grace period
de 24h — decisão fechada em ata ([09:21] Sofia). O FDD, que é o documento "acionável o
suficiente para o dev começar a codar", **nunca especificava esse endpoint** — e
ironicamente já citava a existência dele de passagem ("a secret [...] não é retornada
[...] apenas na criação e na rotação"), sem nunca defini-lo.

As 4 rodadas anteriores (autor × 2, Opus, Fable) tinham todas o mesmo ponto cego:
procuravam por afirmações erradas ou inconsistentes entre si, nunca por promessas de
um documento que o documento seguinte na cadeia (PRD → RFC → ADR → **FDD**) esquece de
cumprir. Esse é um terceiro tipo de falha, distinto dos dois já catalogados nas
Lições 01–03: (1) inventar conteúdo sem fonte, (2) vazar contexto de sessão como se
fosse fato, e agora (3) **omitir silenciosamente** um requisito real ao descer de um
documento de decisão para um documento de implementação. É o mais difícil de pegar
dos três, porque não deixa nenhum texto errado para achar — deixa um vazio, e vazio
não aparece em busca de texto.

## Próximos passos

Nenhum de produção. Entrega final, revisada em 5 camadas (autor, autorrevisão em 2
rodadas, avaliação docente independente em Opus, revisão independente em Fable,
varredura de consistência cruzada em Opus), commitada e publicada no fork público.
