# ADR-005 — Garantia at-least-once com deduplicação via X-Event-Id

## Status

Aceito — decidido em reunião técnica (data exata não informada em `TRANSCRICAO.md`, que só registra "quinta-feira, 09:00"; ver [09:24]–[09:26]).

## Contexto

Com retry (ADR-003), existe a possibilidade real de o mesmo evento ser entregue mais de
uma vez ao cliente (ex.: o worker recebe timeout mas a requisição na verdade foi
processada do lado do cliente antes do timeout). Era preciso decidir o nível de
garantia de entrega e como o cliente lida com duplicatas
(`TRANSCRICAO.md` [09:24] Diego).

## Decisão

O sistema garante **at-least-once delivery**: o cliente pode receber o mesmo evento
mais de uma vez. Cada evento carrega um `event_id` (UUID gerado no momento em que o
evento entra na outbox) enviado no header `X-Event-Id`
(`TRANSCRICAO.md` [09:25] Diego). A responsabilidade de deduplicação fica do lado do
cliente, usando esse `event_id`. Essa responsabilidade deve ser documentada de forma
destacada no portal de desenvolvedor (`TRANSCRICAO.md` [09:26] Marcos).

## Alternativas Consideradas

- **Garantia exactly-once.** Descartada: exigiria coordenação transacional entre os
  dois lados (OMS e sistema do cliente), com complexidade desproporcional ao problema.
  Diego argumentou que at-least-once + `event_id` para dedup do lado do cliente é o
  padrão adotado por provedores de referência de mercado (citou Stripe e GitHub) e
  resolve "99% dos casos" (`TRANSCRICAO.md` [09:25] Diego). Sofia observou que essa
  escolha transfere responsabilidade para o cliente, e a decisão foi mantida mesmo
  assim, com o entendimento de que será documentada claramente
  (`TRANSCRICAO.md` [09:25] Sofia, Diego).

## Consequências

**Positivas**
- Modelo simples de implementar e operar no lado do OMS: não é necessário
  rastrear confirmações de processamento do lado do cliente nem manter estado de
  transação distribuída.
- Alinhado a um padrão amplamente reconhecido no mercado de webhooks, o que facilita a
  integração de clientes já acostumados com esse modelo.

**Negativas**
- Transfere para o cliente a responsabilidade de implementar deduplicação — um cliente
  que não implementar isso corretamente pode processar o mesmo evento (ex.: mesma
  mudança de status) mais de uma vez do lado dele.
- Exige comunicação e documentação explícitas para os clientes (responsabilidade do
  Marcos/PM, via portal de desenvolvedor), sem a qual a garantia é facilmente mal
  compreendida.
