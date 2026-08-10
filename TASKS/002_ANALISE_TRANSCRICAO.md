# TASK 002 — Análise da Transcrição

## Objetivo

Classificar estruturadamente todo o conteúdo de `TRANSCRICAO.md` em requisitos,
decisões, alternativas, restrições, riscos e itens fora de escopo, com timestamp e
participante.

## Contexto

Etapa 2 do fluxo. Reunião de 55 min ([09:00]–[09:53]), 5 participantes: Larissa (Tech
Lead), Marcos (PM), Bruno (Eng. Pleno/Pedidos), Diego (Eng. Sênior/Plataforma, entra às
[09:05]), Sofia (Eng. Segurança, sai às [09:50]).

## Entregáveis

Mapa classificado abaixo — insumo direto para PRD, RFC, ADRs e Tracker.

## Status

🟢 Concluído.

## Decisões fechadas

| # | Decisão | Timestamp/Quem |
| --- | --- | --- |
| 1 | Entrega assíncrona via padrão Outbox no MySQL (não síncrono, não fila externa) | [09:06][09:07] Diego, Larissa |
| 2 | Worker separado em processo (não in-process com a API), polling a cada 2s | [09:09][09:11] Diego, Larissa |
| 3 | Ordering apenas por `order_id`, sem garantia global; limitação documentada | [09:12][09:13] Diego, Larissa |
| 4 | Retry: 5 tentativas, backoff 1m/5m/30m/2h/12h, depois DLQ | [09:15][09:17] Diego, Larissa |
| 5 | DLQ em tabela separada (`webhook_dead_letter`) com payload, motivo, timestamp | [09:18] Diego |
| 6 | Replay manual de DLQ via `POST /admin/webhooks/dead-letter/:id/replay`, role ADMIN obrigatório, com log de auditoria de quem fez o replay | [09:18][09:35][09:36] Diego, Larissa, Sofia |
| 7 | HMAC-SHA256 sobre o corpo, header `X-Signature` | [09:20] Sofia |
| 8 | Secret única por endpoint de webhook (não global) | [09:21] Sofia |
| 9 | Rotação de secret com grace period de 24h (secret antiga válida em paralelo) | [09:21] Sofia |
| 10 | TLS obrigatório (URL deve ser https, senão erro de validação) | [09:23] Sofia |
| 11 | Limite de payload 64KB; se exceder, erro (não trunca) | [09:23][09:24] Sofia, Diego |
| 12 | Garantia at-least-once; dedup por `X-Event-Id` (UUID) do lado do cliente | [09:24][09:25] Diego |
| 13 | Estrutura: `src/modules/webhooks/` seguindo padrão dos demais módulos | [09:27] Bruno |
| 14 | Worker: entry-point `src/worker.ts` + lógica em `src/modules/webhooks/webhook.worker.ts` (ou `webhook.processor.ts`) | [09:28] Bruno |
| 15 | Códigos de erro com prefixo `WEBHOOK_` seguindo padrão `AppError` existente | [09:28][09:29] Bruno, Larissa |
| 16 | Reuso total de Pino, error middleware centralizado, padrão de módulos e schemas Zod | [09:29][09:30] Bruno, Larissa |
| 17 | `PrismaClient` separado por processo no worker (mesmo banco, mesma `DATABASE_URL`) | [09:29][09:30] Diego, Bruno |
| 18 | Filtro de eventos por webhook (lista de status); filtragem ocorre na inserção na outbox, não no envio | [09:33][09:34] Marcos, Bruno, Diego |
| 19 | Endpoint `GET /webhooks/:id/deliveries` — histórico das últimas 100 entregas (payload, response, status, tempo de resposta) | [09:34] Marcos |
| 20 | CRUD de configuração de webhook (`POST`/`PATCH`/`DELETE`/`GET`) exige apenas autenticação normal (não ADMIN) | [09:36] Sofia |
| 21 | `customer_id` vem do body/path, não do JWT (JWT é de usuário operador, não do cliente) | [09:32] Marcos, Larissa |
| 22 | Inserção da outbox dentro da mesma transação de `changeStatus`; se outbox falhar, rollback geral | [09:40][09:41] Bruno, Diego |
| 23 | Função `publishWebhookEvent(tx, order, fromStatus, toStatus)` recebendo o `tx` da transação atual | [09:41] Bruno, Diego |
| 24 | Timeout do HTTP call do worker: 10s | [09:42] Diego |
| 25 | Payload enxuto: `event_id`, `event_type` (`order.status_changed`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents` — sem `items` | [09:43] Diego |
| 26 | Headers do envio: `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type: application/json` | [09:44] Diego, Sofia |
| 27 | ID da outbox é UUID (não auto-incremento), seguindo padrão do projeto | [09:51] Larissa |
| 28 | Payload do evento é renderizado (snapshot) no momento da inserção na outbox, não no momento do envio | [09:52] Larissa, Diego, Bruno |
| 29 | Estimativa: 3 sprints, incluindo revisão de segurança da Sofia (2 dias úteis reservados) | [09:46] Larissa, Sofia |

## Alternativas consideradas e descartadas

| Alternativa | Motivo do descarte | Timestamp/Quem |
| --- | --- | --- |
| Disparo síncrono dentro do `changeStatus` | Cliente lento trava mudança de status de outros pedidos; sem estratégia de rollback se cliente estiver fora do ar | [09:04] Bruno |
| Fila externa (Redis Streams ou similar) | Overengineering para um time pequeno; exigiria subir infraestrutura nova | [09:07] Diego, Larissa |
| Trigger nativo do MySQL para notificar o worker | MySQL não tem `LISTEN/NOTIFY` como Postgres; trigger só executa SQL, não notifica processo externo | [09:09] Diego |
| Retry indefinido com backoff | Evento fica pendurado para sempre se cliente sumir definitivamente | [09:15] Diego |
| 3 tentativas de retry | Muito agressivo; cliente com indisponibilidade de manutenção planejada (2h) já observado historicamente | [09:16] Diego |
| Marcar falha na própria tabela outbox (sem tabela de DLQ separada) | Suja a leitura da outbox principal | [09:18] Diego |
| Truncar payload acima do limite | Preferência por erro explícito quando payload é anormalmente grande | [09:23] Sofia |
| Exactly-once delivery | Exigiria coordenação dos dois lados, complexidade desproporcional; padrão de mercado (Stripe, GitHub) é at-least-once + dedup client-side | [09:25] Diego |

## Questões em aberto / adiadas / fora de escopo

| Item | Classificação | Timestamp/Quem |
| --- | --- | --- |
| Notificação por e-mail ao cliente após falhas consecutivas | Fora de escopo desta fase; possível fase futura | [09:37][09:38] Marcos, Larissa |
| Rate limiting de envio ao cliente (burst de status changes) | Questão em aberto — "observar e decidir depois", não faz parte do escopo atual | [09:38][09:39] Diego, Larissa |
| Dashboard visual para o cliente acompanhar webhooks | Fora de escopo; projeto separado do time de frontend | [09:39][09:40] Marcos, Larissa |
| Escalar para múltiplos workers em paralelo | Fora de escopo; seria necessário particionamento por `order_id` ou lock pessimista no futuro | [09:12][09:13] Diego |
| Arquivamento de eventos entregues (>30 dias) | Fora de escopo desta feature | [09:08] Diego |
| Garantia de ordering global entre pedidos diferentes | Explicitamente não garantida, nem exigida pelos clientes | [09:12][09:14] Diego, Marcos |

## Restrições

- Prazo: fim do trimestre / fim de novembro (cliente Atlas); estimativa interna de 3
  sprints ([09:18] Marcos ameaça migração; [09:45]–[09:47] Marcos, Larissa).
- "Tempo real" definido pelos clientes como <10s de delay ([09:02] Marcos).
- Times pequenos — restrição implícita que motiva a rejeição de Redis/filas externas
  ([09:07] Diego).

## Dependências

- Sofia precisa de ≥2 dias úteis para revisão de segurança antes do deploy ([09:46] Sofia).
- Worker depende do mesmo banco/`DATABASE_URL` da API, mas processo Node separado
  ([09:11][09:29] Diego, Bruno).

## Riscos citados na reunião

- Vazamento de secret em log de aplicação do cliente (já ocorreu antes) — motiva
  rotação com grace period ([09:22] Diego).
- Cliente indisponível por período longo (2h+ observado) — motiva 5 tentativas em vez
  de 3 ([09:16] Diego).

## Itens concluídos

Classificação completa acima.

## Próximos passos

`TASKS/003_ANALISE_CODIGO.md` (consolidação com o mapa de código de TASKS/001) e início
dos ADRs.

## Observações

Nenhum requisito foi inferido além do que está literalmente na transcrição. Onde a
reunião cita analogia de mercado (Stripe/GitHub em [09:25]), a citação é preservada
como justificativa dita por Diego, não como fato assumido pela IA.
