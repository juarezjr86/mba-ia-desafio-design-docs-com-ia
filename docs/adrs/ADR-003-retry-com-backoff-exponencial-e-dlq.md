# ADR-003 — Retry com backoff exponencial (5 tentativas) e Dead Letter Queue

## Status

Aceito — decidido em reunião técnica (data exata não informada em `TRANSCRICAO.md`, que só registra "quinta-feira, 09:00"; ver [09:14]–[09:18]).

## Contexto

Endpoints de clientes podem estar temporariamente indisponíveis (ex.: manutenção
planejada). Era preciso decidir quantas vezes retentar a entrega de um evento, com que
intervalo, e o que fazer com um evento que esgota as tentativas
(`TRANSCRICAO.md` [09:14] Larissa).

## Decisão

Retry com **backoff exponencial em 5 tentativas**, com intervalos **1 minuto, 5
minutos, 30 minutos, 2 horas, 12 horas** (janela total de ~15h entre a primeira falha e
a última tentativa) (`TRANSCRICAO.md` [09:15][09:17] Diego, Larissa). Esgotadas as 5
tentativas, o evento é movido para uma **tabela separada `webhook_dead_letter`**,
guardando payload, motivo da falha e timestamp (`TRANSCRICAO.md` [09:18] Diego).

Reprocessamento é **manual**, via endpoint administrativo
`POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como
pendente (`TRANSCRICAO.md` [09:18] Diego). Esse endpoint exige role `ADMIN`
(reaproveitando `requireRole` de `src/middlewares/auth.middleware.ts`) e deve logar
quem executou o replay, para auditoria (`TRANSCRICAO.md` [09:35][09:36] Larissa, Sofia).

O timeout de cada chamada HTTP do worker ao endpoint do cliente é de **10 segundos**;
não responder nesse prazo é tratado como falha e entra no fluxo de retry
(`TRANSCRICAO.md` [09:42] Diego).

## Alternativas Consideradas

- **Retry indefinido com backoff.** Descartada: um evento pode ficar pendurado para
  sempre se o cliente sumir definitivamente, sem sinalização clara de falha permanente
  (`TRANSCRICAO.md` [09:15] Diego).
- **3 tentativas (mais agressivo).** Descartada: Bruno propôs 3, mas Diego argumentou
  que 3 tentativas em uma janela curta (~30min) mataria a entrega durante uma
  indisponibilidade de manutenção planejada já observada em cliente real, de até 2
  horas (`TRANSCRICAO.md` [09:16] Bruno, Diego).
- **Marcar falha permanente na própria tabela `webhook_outbox`** (sem tabela de DLQ
  separada). Descartada: sujaria a leitura da fila principal de pendentes; uma tabela
  separada fica mais limpa e serve como evidência para debug e reprocessamento
  (`TRANSCRICAO.md` [09:18] Diego, Bruno).

## Consequências

**Positivas**
- Tolerância a indisponibilidades reais de cliente já observadas historicamente, sem
  descartar eventos prematuramente.
- Separação clara entre "fila de trabalho" (`webhook_outbox`) e "eventos que precisam
  de intervenção humana" (`webhook_dead_letter`), facilitando operação e observabilidade.
- Reprocessamento controlado e auditável, evitando reenvio acidental em massa.

**Negativas**
- Um evento pode demorar até ~15 horas para ser definitivamente classificado como
  falha, mantendo estado "pendente" por um período longo do ponto de vista do cliente
  final.
- Reprocessamento manual implica que ninguém é automaticamente notificado quando um
  evento cai em DLQ nesta fase — não há alerta por e-mail (explicitamente adiado, ver
  RFC, Questões em Aberto). Depende de alguém consultar a DLQ proativamente.
- Mais uma tabela e mais um endpoint administrativo para manter e proteger
  (mitigado por reaproveitar `requireRole('ADMIN')` já existente).
