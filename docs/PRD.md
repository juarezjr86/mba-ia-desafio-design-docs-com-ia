# PRD — Sistema de Webhooks de Notificação de Pedidos

## Resumo e contexto da feature

O OMS vai passar a notificar sistemas de clientes B2B automaticamente quando o status
de um pedido muda, via webhooks outbound (o OMS envia, não recebe —
`TRANSCRICAO.md` [09:02][09:03] Marcos, Sofia). A decisão técnica foi fechada em
reunião entre Tech Lead, PM, Engenharia e Segurança; este documento consolida a visão
de produto sobre essa feature, com base na proposta técnica do [RFC](./RFC.md), nas
decisões registradas nos [ADRs](./adrs/README.md) e no detalhamento do [FDD](./FDD.md).

## Problema e motivação

Clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) hoje descobrem mudança de
status de pedido fazendo polling manual em `GET /orders`, o que a Atlas descreveu como
lento e caro de manter do lado deles. A Atlas sinalizou risco de migrar para um
concorrente caso a feature não seja entregue até o fim do trimestre
(`TRANSCRICAO.md` [09:00] Marcos).

## Público-alvo e cenários de uso

- **Cliente B2B integrador** (ex.: Atlas, MaxDistribuição, Nova Cargo): cadastra um
  endpoint HTTPS para receber notificações, escolhe quais status quer acompanhar, e
  recebe eventos assim que o pedido dele muda de status, sem precisar consultar a API
  repetidamente.
- **Operador interno do OMS** (usuário autenticado via JWT do sistema): cadastra e
  gerencia webhooks em nome do customer, via API (não existe painel — a integração é
  direto pela API, `TRANSCRICAO.md` [09:32] Marcos).
- **Administrador (role `ADMIN`)**: investiga falhas de entrega e reprocessa eventos
  presos em Dead Letter Queue.

## Objetivos e métricas de sucesso

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| Eliminar a necessidade de polling manual dos clientes | Latência entre mudança de status e entrega bem-sucedida do evento | Abaixo de 10 segundos no caso de sucesso na 1ª tentativa (definição de "tempo real" dada pelos próprios clientes — `TRANSCRICAO.md` [09:02] Marcos) |
| Não perder eventos de negócio | Eventos perdidos silenciosamente (sem chegar a `delivered` nem a `dead_letter`) | Zero — todo evento inserido na outbox termina em `delivered` ou em `webhook_dead_letter`, nunca desaparece sem rastro |
| Reter a Atlas e demais clientes B2B que pediram a feature | Entrega dentro do prazo comunicado (fim de novembro / fim do trimestre) | Entrega em até 3 sprints, incluindo revisão de segurança (`TRANSCRICAO.md` [09:46][09:47] Larissa, Marcos) |

## Escopo

- CRUD de configuração de webhook por customer (URL, secret, status inscritos).
- Emissão de evento de mudança de status via padrão Outbox, atômica com a transação
  de `changeStatus`.
- Entrega assíncrona via worker dedicado, com retry, backoff e DLQ.
- Autenticação de payload via HMAC-SHA256 com secret por endpoint, rotacionável.
- Garantia at-least-once com deduplicação via `X-Event-Id` do lado do cliente.
- Histórico de entregas por webhook (`GET /webhooks/:id/deliveries`).
- Reprocessamento manual de eventos em DLQ, restrito a `ADMIN`.

## Fora de escopo

- **Notificação por e-mail ao cliente após falhas consecutivas de entrega.**
  Explicitamente descartada para esta fase; possível fase futura, após medir o
  impacto real (`TRANSCRICAO.md` [09:37][09:38] Marcos, Larissa).
- **Dashboard visual para o cliente acompanhar seus webhooks.** Fora de escopo; seria
  um projeto separado do time de frontend — esta entrega é somente API
  (`TRANSCRICAO.md` [09:39][09:40] Marcos, Larissa).
- **Rate limiting de envio ao cliente** em caso de burst de mudanças de status. Ficou
  registrado como "observar e decidir depois", não faz parte desta entrega
  (`TRANSCRICAO.md` [09:38][09:39] Diego, Larissa).
- **Escalonamento para múltiplos workers em paralelo.** Adiado — exigiria
  particionamento por `order_id` ou lock pessimista para preservar ordering
  (`TRANSCRICAO.md` [09:12][09:13] Diego, Bruno).
- **Garantia de ordering global** entre pedidos diferentes. Não é objetivo desta
  feature; os clientes nunca pediram essa garantia, apenas saber quando cada pedido
  deles muda (`TRANSCRICAO.md` [09:12]–[09:14] Diego, Marcos).
- **Arquivamento de eventos entregues.** Fora do escopo desta feature
  (`TRANSCRICAO.md` [09:08] Diego).

## Requisitos funcionais

1. O cliente (via usuário autenticado do OMS) pode cadastrar um webhook informando
   `url` e a lista de status de pedido que deseja acompanhar; a secret é gerada pelo
   servidor e devolvida na criação (`TRANSCRICAO.md` [09:31] Marcos).
2. O sistema recusa o cadastro de webhook com URL que não seja HTTPS
   (`TRANSCRICAO.md` [09:23] Sofia).
3. O cliente pode editar (`PATCH`) a URL e/ou a lista de status inscritos de um
   webhook já cadastrado (`TRANSCRICAO.md` [09:33] Bruno).
4. O cliente pode remover (`DELETE`) um webhook cadastrado (`TRANSCRICAO.md` [09:33]
   Bruno).
5. O cliente pode listar (`GET`) os webhooks cadastrados de um customer
   (`TRANSCRICAO.md` [09:33] Bruno).
6. Quando um pedido muda de status, o sistema insere um evento na outbox apenas para
   os webhooks daquele customer inscritos naquele status específico — filtragem ocorre
   na inserção, não no envio (`TRANSCRICAO.md` [09:33][09:34] Marcos, Bruno, Diego).
7. O sistema entrega o evento ao endpoint do cliente assinado com HMAC-SHA256 (header
   `X-Signature`), permitindo ao cliente validar autenticidade e integridade do
   payload (`TRANSCRICAO.md` [09:20] Sofia).
8. Em caso de falha na entrega, o sistema retenta automaticamente até 5 vezes com
   backoff exponencial (1m/5m/30m/2h/12h) antes de mover o evento para uma fila de
   Dead Letter (`TRANSCRICAO.md` [09:15]–[09:18] Diego).
9. O cliente pode consultar o histórico das últimas 100 entregas de um webhook,
   incluindo sucesso/falha, payload, resposta e tempo de resposta
   (`TRANSCRICAO.md` [09:34] Marcos).
10. Um usuário com role `ADMIN` pode reprocessar manualmente um evento em Dead Letter
    via endpoint dedicado, e essa ação é registrada para auditoria
    (`TRANSCRICAO.md` [09:18][09:35][09:36] Diego, Larissa, Sofia).
11. O cliente pode solicitar a rotação da secret de um webhook; a secret antiga
    permanece válida por 24h em paralelo à nova (`TRANSCRICAO.md` [09:21] Sofia).
12. Cada evento entregue carrega um identificador único (`X-Event-Id`) que permite ao
    cliente deduplicar entregas repetidas do mesmo evento
    (`TRANSCRICAO.md` [09:25] Diego).

## Requisitos não funcionais

- Latência-alvo de entrega abaixo de 10 segundos no caminho feliz
  (`TRANSCRICAO.md` [09:02][09:10] Marcos, Larissa).
- Timeout de 10 segundos por chamada HTTP a endpoint de cliente
  (`TRANSCRICAO.md` [09:42] Diego).
- Limite de 64KB no tamanho do payload de evento; excedido esse limite, o envio falha
  explicitamente em vez de truncar (`TRANSCRICAO.md` [09:23][09:24] Sofia, Diego).
- Toda comunicação de webhook com o cliente deve ser sobre HTTPS
  (`TRANSCRICAO.md` [09:23] Sofia).
- Garantia de entrega at-least-once, nunca exactly-once
  (`TRANSCRICAO.md` [09:24][09:25] Diego).
- A feature não pode degradar a robustez da transação `changeStatus` já existente —
  falha de infraestrutura de webhook não pode deixar o pedido em estado inconsistente
  (`TRANSCRICAO.md` [09:40][09:41] Bruno, Diego).
- Reuso de logging (Pino) e tratamento de erro (`AppError`/error middleware) já
  existentes, sem introduzir ferramentas novas de observabilidade
  (`TRANSCRICAO.md` [09:29] Bruno).

## Decisões e trade-offs principais

Ver [RFC — Proposta técnica](./RFC.md#proposta-técnica) e os 6 ADRs em
`docs/adrs/`: padrão Outbox (vs. síncrono/fila externa), worker dedicado com polling
de 2s (vs. trigger de banco), retry de 5 tentativas com DLQ (vs. retry indefinido ou 3
tentativas), HMAC-SHA256 com secret por endpoint (vs. secret global), at-least-once
com `X-Event-Id` (vs. exactly-once), reuso máximo dos padrões de código já existentes.

## Dependências

- MySQL 8 já provisionado (`docker-compose.yml`) e `PrismaClient` já usado pelo
  projeto.
- Revisão de segurança da Sofia antes do deploy — pelo menos 2 dias úteis reservados,
  focada em HMAC e geração de secret (`TRANSCRICAO.md` [09:46] Sofia).
- Comunicação dos clientes (Atlas, MaxDistribuição, Nova Cargo) pelo time de produto
  sobre o novo mecanismo e o contrato de deduplicação — responsabilidade do Marcos, via
  portal de desenvolvedor (`TRANSCRICAO.md` [09:26][09:40][09:47][09:49] Marcos).

## Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Vazamento de secret do lado do cliente (já ocorreu antes com outro cliente) | Média — já observado historicamente (`TRANSCRICAO.md` [09:22] Diego) | Alto — comprometeria a integridade dos eventos daquele cliente | Secret única por endpoint (não global) + rotação sob demanda com grace period de 24h ([ADR-004](./adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)) |
| Cliente indisponível por período prolongado (indisponibilidade de manutenção de até 2h já observada) | Média | Médio — atraso na notificação, não perda, dado o retry de 5 tentativas em até ~15h | Backoff exponencial dimensionado para cobrir janelas de indisponibilidade conhecidas ([ADR-003](./adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)) |
| Worker (processo separado) fica fora do ar e para de processar a outbox | Baixa a média — depende de operação/deploy | Alto — atraso generalizado na entrega de todos os eventos pendentes | Processo isolado da API, reiniciável independentemente; eventos ficam retidos como `pending`, sem perda ([ADR-002](./adrs/ADR-002-worker-separado-com-polling.md)) |
| Cliente não implementa deduplicação corretamente e processa evento duplicado | Média — depende da qualidade da integração do cliente | Médio — efeito colateral do lado do cliente, fora do controle direto do OMS | Documentação clara do contrato at-least-once + `X-Event-Id` no portal de desenvolvedor (`TRANSCRICAO.md` [09:26] Marcos) |

## Critérios de aceitação

- Ao mudar o status de um pedido para um valor que algum webhook do customer está
  inscrito, um evento é entregue ao endpoint correspondente em condições normais de
  rede, assinado com HMAC válido e com todos os headers definidos no FDD.
- Nenhuma mudança de status é perdida ou corrompida por falha no subsistema de
  webhooks — a transação de `changeStatus` só é efetivada se a inserção na outbox
  também for bem-sucedida.
- Falha de entrega segue exatamente a progressão de retry (5 tentativas,
  1m/5m/30m/2h/12h) antes de cair em DLQ.
- Endpoint de replay de DLQ só aceita chamadas de usuários com role `ADMIN` e a ação
  fica registrada em log.
- Nenhum requisito descartado ou adiado na reunião (e-mail de alerta, dashboard,
  rate limiting de saída, ordering global, múltiplos workers) aparece implementado ou
  como requisito obrigatório nesta entrega.

## Estratégia de testes e validação

- Testes de integração ponta a ponta seguindo o padrão já usado em
  `tests/orders.test.ts` (Vitest + Supertest + app real via `getTestApp()`,
  `tests/helpers/factories.ts`), cobrindo: criação de webhook, filtragem de eventos por
  status inscrito, inserção atômica na outbox junto com `changeStatus`, e resposta de
  erro para URL não-HTTPS.
- Testes de unidade para o cálculo de HMAC e para a lógica de progressão de backoff
  (tempos e número de tentativas), isolados da camada HTTP.
- Validação manual do fluxo de retry/DLQ simulando indisponibilidade do endpoint de
  destino (mock HTTP retornando timeout/erro) antes do deploy.
- Revisão de segurança dedicada da Sofia sobre a geração e validação de HMAC e sobre a
  geração/rotação de secret, como gate antes de subir para produção
  (`TRANSCRICAO.md` [09:46] Sofia).
- Estes testes de exemplo não foram implementados nesta entrega — a entrega é
  puramente documental; a estratégia acima descreve o plano de testes para quando a
  implementação for iniciada.
