# ADR-004 — Autenticação de webhook via HMAC-SHA256 com secret por endpoint

## Status

Aceito — decidido em reunião técnica (data exata não informada em `TRANSCRICAO.md`, que só registra "quinta-feira, 09:00"; ver [09:19]–[09:24]).

## Contexto

Os eventos de webhook expõem dados de pedidos para sistemas fora da infraestrutura do
OMS. Sofia (Engenharia de Segurança) levantou que o cliente precisa conseguir validar
que a requisição realmente veio do OMS e que o payload não foi adulterado em trânsito
(`TRANSCRICAO.md` [09:19] Sofia).

## Decisão

Assinar cada requisição de webhook com **HMAC-SHA256** sobre o corpo (payload) da
requisição, enviando a assinatura no header `X-Signature`
(`TRANSCRICAO.md` [09:20] Sofia). Cada **endpoint de webhook cadastrado tem uma secret
própria e única** (não uma secret global da plataforma) — a tabela de configuração de
webhook armazena `url`, `secret`, `customer_id` e estado ativo
(`TRANSCRICAO.md` [09:21] Sofia, Bruno).

A secret é **rotacionável**: o cliente pode solicitar uma nova secret via API; a
secret antiga permanece válida em paralelo por um **grace period de 24 horas** para
permitir a migração dos sistemas do cliente, e depois disso é invalidada
(`TRANSCRICAO.md` [09:21] Sofia). Essa decisão foi motivada por um incidente real
anterior de vazamento de secret em log de aplicação de cliente
(`TRANSCRICAO.md` [09:22] Diego).

Requisitos complementares de segurança, tratados como validação de schema (não como
decisões arquiteturais separadas): TLS obrigatório — a URL do webhook cadastrada deve
ser `https`, senão o cadastro é recusado com erro de validação
(`TRANSCRICAO.md` [09:23] Sofia); e limite de **64KB** no tamanho do payload — se
excedido, o envio falha explicitamente (não trunca) (`TRANSCRICAO.md` [09:23][09:24]
Sofia, Diego).

## Alternativas Consideradas

- **Secret global única para toda a plataforma.** Descartada: Sofia apontou que, se
  essa secret vazar, todos os clientes ficam comprometidos simultaneamente — a secret
  precisa ser por endpoint (`TRANSCRICAO.md` [09:21] Sofia).
- **Truncar o payload quando exceder o limite de tamanho**, em vez de recusar o envio.
  Descartada: Sofia defendeu erro explícito — payload anormalmente grande é sinal de
  algo errado no sistema, e truncar mascararia esse problema
  (`TRANSCRICAO.md` [09:23] Sofia).

## Consequências

**Positivas**
- Blast radius de um vazamento de secret limitado a um único cliente/endpoint, não à
  plataforma inteira.
- Cliente consegue verificar autenticidade e integridade do payload de forma padrão de
  mercado (HMAC-SHA256), sem exigir infraestrutura extra do lado dele.
- Rotação com grace period evita downtime de integração durante troca de secret,
  reduzindo o incentivo a manter uma secret comprometida por medo de quebrar a
  integração.

**Negativas**
- Aumenta a superfície de configuração por endpoint (mais um segredo por cliente para
  gerar, armazenar com segurança e rotacionar).
- Durante o grace period de 24h, duas secrets ficam simultaneamente válidas para o
  mesmo endpoint — janela em que uma secret comprometida ainda funciona, por desenho.
- A validação de HTTPS e tamanho de payload, embora simples, precisa ser reforçada
  tanto no cadastro (schema Zod) quanto no momento do envio pelo worker, para não haver
  brecha caso a URL seja alterada por outro caminho.
