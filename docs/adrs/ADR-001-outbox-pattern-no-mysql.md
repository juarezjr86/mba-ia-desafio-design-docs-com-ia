# ADR-001 — Outbox Pattern no MySQL para emissão de eventos de webhook

## Status

Aceito — decidido em reunião técnica (data exata não informada em `TRANSCRICAO.md`, que só registra "quinta-feira, 09:00"; ver [09:06]–[09:08]).

## Contexto

O OMS precisa notificar sistemas de clientes B2B (Atlas Comercial, MaxDistribuição,
Nova Cargo) quando o status de um pedido muda, em até 10 segundos (definição de "tempo
real" dada pelos próprios clientes — `TRANSCRICAO.md` [09:02] Marcos).

A mudança de status hoje é uma transação já pesada
(`src/modules/orders/order.service.ts`, método `changeStatus`): atualiza `orders`,
insere em `order_status_history` e debita/repõe `stock_quantity` de produtos. Bruno
apontou em [09:04] que inserir uma chamada HTTP síncrona nesse fluxo bloquearia a
mudança de status de outros pedidos caso o cliente destino esteja lento, e que não há
como fazer rollback de uma mudança de status já efetivada caso o cliente esteja fora do
ar. Era preciso desacoplar a emissão do evento da entrega do evento, sem perder a
garantia de que todo `changeStatus` bem-sucedido gera exatamente um evento.

## Decisão

Adotar o padrão **Transactional Outbox** sobre o MySQL já usado pelo projeto (não um
banco novo, não uma fila externa). Dentro da mesma transação SQL de `changeStatus`
(`this.prisma.$transaction` em `src/modules/orders/order.service.ts`), inserir uma
linha na tabela `webhook_outbox` com o payload do evento já renderizado (ver ADR-002 e
FDD para o fluxo completo). Se a transação principal faz commit, o evento foi
persistido; se faz rollback, o evento desaparece junto — nenhuma inconsistência
possível entre estado do pedido e evento emitido (`TRANSCRICAO.md` [09:06][09:40][09:41]
Diego, Bruno).

Um processo separado (worker, ver ADR-002) lê a tabela `webhook_outbox` de forma
assíncrona e entrega os eventos.

## Alternativas Consideradas

- **Disparo HTTP síncrono dentro de `changeStatus`.** Descartada: acopla a latência/
  disponibilidade de um sistema externo à transação crítica de negócio; sem estratégia
  de rollback viável se o cliente estiver indisponível (`TRANSCRICAO.md` [09:04] Bruno).
- **Fila externa dedicada (ex: Redis Streams).** Descartada: exigiria subir e operar
  infraestrutura nova para um time pequeno; Diego chamou de "overengineering" para o
  problema em questão (`TRANSCRICAO.md` [09:07] Diego, Larissa).

## Consequências

**Positivas**
- Garantia forte de atomicidade entre estado do pedido e emissão do evento, sem
  infraestrutura adicional — reaproveita o MySQL e o `PrismaClient` já usados pelo
  projeto (`prisma/schema.prisma`).
- Reduz o acoplamento entre a API e a disponibilidade dos endpoints dos clientes.

**Negativas**
- Introduz latência mínima adicional (o evento só é entregue quando o worker o lê —
  ver ADR-002 para o trade-off de polling).
- Exige nova tabela e nova responsabilidade operacional (limpeza/arquivamento de
  eventos entregues), explicitamente colocada fora do escopo desta feature
  (`TRANSCRICAO.md` [09:08] Diego).
- Acopla a modelagem da outbox ao mesmo banco da aplicação principal — um pico de
  volume de eventos compete por recursos de I/O com as tabelas de negócio.
