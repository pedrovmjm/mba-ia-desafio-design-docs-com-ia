# Tracker de Rastreabilidade

Este tracker liga requisitos, decisões, restrições e contratos às duas fontes
permitidas: a reunião em [`TRANSCRICAO.md`](../TRANSCRICAO.md) e arquivos reais
do código. Exemplos de payload, nomes internos e formas HTTP no FDD concretizam
as decisões rastreadas abaixo; não criam novo escopo de produto.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-OBJ-01 | `docs/PRD.md` | Objetivo/Métrica | p95 de entrega menor que 10 segundos | TRANSCRICAO | `[09:02] Marcos` |
| PRD-OBJ-02 | `docs/PRD.md` | Objetivo/Métrica | Todo evento commitado termina entregue ou em DLQ | TRANSCRICAO | `[09:06] Diego`; `[09:18] Diego`; `[09:40] Bruno` |
| PRD-OBJ-03 | `docs/PRD.md` | Objetivo | Atender os três clientes solicitantes | TRANSCRICAO | `[09:00] Marcos` |
| PRD-OBJ-04 | `docs/PRD.md` | Prazo | Entrega estimada em três sprints | TRANSCRICAO | `[09:46] Larissa` |
| PRD-OBJ-05 | `docs/PRD.md` | Segurança | Reservar dois dias úteis para revisão | TRANSCRICAO | `[09:46] Sofia` |
| PRD-OOS-01 | `docs/PRD.md` | Fora de escopo | E-mail de alerta adiado | TRANSCRICAO | `[09:37] Larissa`; `[09:38] Marcos` |
| PRD-OOS-02 | `docs/PRD.md` | Fora de escopo | Dashboard visual adiado | TRANSCRICAO | `[09:39] Marcos`; `[09:40] Larissa` |
| PRD-OOS-03 | `docs/PRD.md` | Fora de escopo | Rate limiting apenas será observado | TRANSCRICAO | `[09:38] Diego`; `[09:39] Larissa` |
| PRD-OOS-04 | `docs/PRD.md` | Fora de escopo | Arquivamento da outbox não entra | TRANSCRICAO | `[09:08] Diego` |
| PRD-OOS-05 | `docs/PRD.md` | Fora de escopo | Múltiplos workers adiados | TRANSCRICAO | `[09:12] Diego`; `[09:13] Diego` |
| PRD-OOS-06 | `docs/PRD.md` | Limitação | Sem ordering global | TRANSCRICAO | `[09:13] Larissa`; `[09:14] Marcos` |
| PRD-OOS-07 | `docs/PRD.md` | Fora de escopo | Webhooks inbound excluídos | TRANSCRICAO | `[09:02] Marcos`; `[09:03] Sofia` |
| PRD-OOS-08 | `docs/PRD.md` | Fora de escopo | Itens não entram no payload | TRANSCRICAO | `[09:43] Diego`; `[09:44] Bruno` |
| PRD-RF-01 | `docs/PRD.md` | Requisito Funcional | Cadastrar endpoint e gerar secret | TRANSCRICAO | `[09:31] Marcos` |
| PRD-RF-02 | `docs/PRD.md` | Requisito Funcional | Editar endpoint | TRANSCRICAO | `[09:33] Bruno` |
| PRD-RF-03 | `docs/PRD.md` | Requisito Funcional | Remover e listar endpoints do customer | TRANSCRICAO | `[09:33] Bruno` |
| PRD-RF-04 | `docs/PRD.md` | Requisito Funcional | Filtrar status na inserção da outbox | TRANSCRICAO | `[09:33] Marcos`; `[09:34] Bruno` |
| PRD-RF-05 | `docs/PRD.md` | Requisito Funcional | Criar snapshot na mesma transação | TRANSCRICAO | `[09:40] Bruno`; `[09:52] Larissa` |
| PRD-RF-06 | `docs/PRD.md` | Requisito Funcional | Payload enxuto de mudança de status | TRANSCRICAO | `[09:43] Diego` |
| PRD-RF-07 | `docs/PRD.md` | Requisito Funcional | Headers de entrega acordados | TRANSCRICAO | `[09:44] Diego`; `[09:44] Sofia` |
| PRD-RF-08 | `docs/PRD.md` | Requisito Funcional | Rotação com grace period de 24h | TRANSCRICAO | `[09:21] Sofia`; `[09:22] Sofia` |
| PRD-RF-09 | `docs/PRD.md` | Requisito Funcional | Histórico de entregas | TRANSCRICAO | `[09:34] Marcos` |
| PRD-RF-10 | `docs/PRD.md` | Requisito Funcional | Cinco retries e DLQ | TRANSCRICAO | `[09:15] Diego`; `[09:17] Diego`; `[09:18] Diego` |
| PRD-RF-11 | `docs/PRD.md` | Requisito Funcional | Replay somente ADMIN e auditado | TRANSCRICAO | `[09:35] Larissa`; `[09:36] Sofia` |
| PRD-RF-12 | `docs/PRD.md` | Requisito Funcional | CRUD para role autenticada e customer no contrato | TRANSCRICAO | `[09:32] Larissa`; `[09:37] Sofia` |
| PRD-RNF-01 | `docs/PRD.md` | Requisito Não Funcional | Menos de 10s e polling de 2s | TRANSCRICAO | `[09:02] Marcos`; `[09:09] Diego`; `[09:10] Larissa` |
| PRD-RNF-02 | `docs/PRD.md` | Requisito Não Funcional | Entrega assíncrona em processo separado | TRANSCRICAO | `[09:04] Bruno`; `[09:11] Diego` |
| PRD-RNF-03 | `docs/PRD.md` | Requisito Não Funcional | Atomicidade entre pedido e outbox | TRANSCRICAO | `[09:06] Diego`; `[09:40] Bruno`; `[09:41] Diego` |
| PRD-RNF-04 | `docs/PRD.md` | Requisito Não Funcional | At-least-once e deduplicação | TRANSCRICAO | `[09:24] Diego`; `[09:25] Diego` |
| PRD-RNF-05 | `docs/PRD.md` | Requisito Não Funcional | HMAC por endpoint e TLS | TRANSCRICAO | `[09:20] Sofia`; `[09:21] Sofia`; `[09:23] Sofia` |
| PRD-RNF-06 | `docs/PRD.md` | Requisito Não Funcional | Timeout de 10 segundos | TRANSCRICAO | `[09:42] Diego` |
| PRD-RNF-07 | `docs/PRD.md` | Requisito Não Funcional | Limite de 64 KiB sem truncar | TRANSCRICAO | `[09:23] Sofia`; `[09:24] Diego`; `[09:24] Larissa` |
| PRD-RNF-08 | `docs/PRD.md` | Requisito Não Funcional | Ordering por pedido no single-worker | TRANSCRICAO | `[09:12] Diego`; `[09:13] Larissa` |
| PRD-RNF-09 | `docs/PRD.md` | Requisito Não Funcional | Reuso dos padrões do OMS | TRANSCRICAO | `[09:27] Bruno`; `[09:29] Bruno`; `[09:30] Larissa` |
| PRD-RSK-01 | `docs/PRD.md` | Risco | Evento fora da transação | TRANSCRICAO | `[09:40] Bruno`; `[09:41] Diego` |
| PRD-RSK-02 | `docs/PRD.md` | Risco | Backlog com worker parado ou clientes offline | TRANSCRICAO | `[09:07] Bruno`; `[09:08] Diego`; `[09:11] Diego` |
| PRD-RSK-03 | `docs/PRD.md` | Risco | Vazamento de secret | TRANSCRICAO | `[09:21] Sofia`; `[09:22] Diego` |
| PRD-RSK-04 | `docs/PRD.md` | Risco | Cliente processar duplicata | TRANSCRICAO | `[09:24] Diego`; `[09:25] Sofia`; `[09:26] Marcos` |
| PRD-RSK-05 | `docs/PRD.md` | Risco | Atraso e churn da Atlas | TRANSCRICAO | `[09:00] Marcos`; `[09:45] Marcos`; `[09:46] Larissa` |
| RFC-PROP-01 | `docs/RFC.md` | Proposta | Outbox no MySQL existente | TRANSCRICAO | `[09:06] Diego`; `[09:07] Diego`; `[09:08] Larissa` |
| RFC-PROP-02 | `docs/RFC.md` | Proposta | Worker separado com polling | TRANSCRICAO | `[09:09] Diego`; `[09:11] Diego`; `[09:11] Larissa` |
| RFC-PROP-03 | `docs/RFC.md` | Proposta | HMAC, secret por endpoint e TLS | TRANSCRICAO | `[09:20] Sofia`; `[09:21] Sofia`; `[09:23] Sofia` |
| RFC-PROP-04 | `docs/RFC.md` | Proposta | Retry, DLQ e at-least-once | TRANSCRICAO | `[09:15] Diego`; `[09:18] Diego`; `[09:24] Diego` |
| RFC-ALT-01 | `docs/RFC.md` | Alternativa Descartada | HTTP síncrono no `changeStatus` | TRANSCRICAO | `[09:03] Larissa`; `[09:04] Bruno`; `[09:06] Diego` |
| RFC-ALT-02 | `docs/RFC.md` | Alternativa Descartada | Redis Streams/fila externa | TRANSCRICAO | `[09:07] Larissa`; `[09:07] Diego` |
| RFC-ALT-03 | `docs/RFC.md` | Alternativa Descartada | Trigger MySQL | TRANSCRICAO | `[09:09] Bruno`; `[09:09] Diego` |
| RFC-ALT-04 | `docs/RFC.md` | Alternativa Descartada | Worker dentro da API | TRANSCRICAO | `[09:11] Diego` |
| RFC-ALT-05 | `docs/RFC.md` | Alternativa Descartada | Somente três retentativas | TRANSCRICAO | `[09:16] Bruno`; `[09:16] Diego` |
| RFC-ALT-06 | `docs/RFC.md` | Alternativa Descartada | Retry indefinido | TRANSCRICAO | `[09:15] Diego` |
| RFC-ALT-07 | `docs/RFC.md` | Alternativa Descartada | Exactly-once | TRANSCRICAO | `[09:25] Diego` |
| RFC-ALT-08 | `docs/RFC.md` | Alternativa Descartada | Secret global | TRANSCRICAO | `[09:21] Sofia` |
| RFC-QA-01 | `docs/RFC.md` | Questão em Aberto | Rate limiting de saída | TRANSCRICAO | `[09:38] Diego`; `[09:39] Larissa` |
| RFC-QA-02 | `docs/RFC.md` | Questão em Aberto | Escala e ordering com múltiplos workers | TRANSCRICAO | `[09:12] Diego`; `[09:13] Diego`; `[09:13] Larissa` |
| RFC-QA-03 | `docs/RFC.md` | Questão em Aberto | Política de arquivamento | TRANSCRICAO | `[09:08] Diego` |
| RFC-QA-04 | `docs/RFC.md` | Questão em Aberto | Endurecimento futuro da autorização | TRANSCRICAO | `[09:36] Larissa`; `[09:37] Sofia` |
| FDD-FL-01 | `docs/FDD.md` | Fluxo | Publisher recebe a transação e filtra endpoints | TRANSCRICAO | `[09:34] Bruno`; `[09:40] Bruno`; `[09:41] Bruno` |
| FDD-FL-02 | `docs/FDD.md` | Fluxo | Worker consulta lote antigo a cada 2s | TRANSCRICAO | `[09:08] Diego`; `[09:09] Diego` |
| FDD-FL-03 | `docs/FDD.md` | Fluxo | Timeout e classificação de falha de entrega | TRANSCRICAO | `[09:14] Larissa`; `[09:42] Sofia`; `[09:42] Diego` |
| FDD-FL-04 | `docs/FDD.md` | Fluxo | Cinco retentativas após o envio inicial | TRANSCRICAO | `[09:15] Diego`; `[09:17] Diego`; `[09:48] Larissa` |
| FDD-FL-05 | `docs/FDD.md` | Fluxo | Movimentação para DLQ e replay | TRANSCRICAO | `[09:18] Diego`; `[09:35] Diego`; `[09:36] Sofia` |
| FDD-FL-06 | `docs/FDD.md` | Fluxo | Rotação mantém duas secrets válidas por 24h | TRANSCRICAO | `[09:21] Sofia`; `[09:22] Sofia` |
| FDD-DATA-01 | `docs/FDD.md` | Modelo de Dados | Endpoint contém URL, secret, customer, ativo e eventos | TRANSCRICAO | `[09:21] Bruno`; `[09:31] Marcos`; `[09:33] Marcos` |
| FDD-DATA-02 | `docs/FDD.md` | Modelo de Dados | Outbox indexada por status e criação | TRANSCRICAO | `[09:08] Diego` |
| FDD-DATA-03 | `docs/FDD.md` | Modelo de Dados | Delivery armazena payload, resposta e duração | TRANSCRICAO | `[09:34] Marcos` |
| FDD-DATA-04 | `docs/FDD.md` | Modelo de Dados | DLQ separada guarda payload, motivo e horário | TRANSCRICAO | `[09:17] Larissa`; `[09:18] Diego` |
| FDD-DATA-05 | `docs/FDD.md` | Modelo de Dados | IDs em UUID | TRANSCRICAO | `[09:51] Larissa` |
| FDD-API-01 | `docs/FDD.md` | Contrato HTTP | `POST /api/v1/webhooks` | TRANSCRICAO | `[09:31] Marcos`; `[09:32] Larissa` |
| FDD-API-02 | `docs/FDD.md` | Contrato HTTP | `GET /api/v1/webhooks` por customer | TRANSCRICAO | `[09:33] Bruno` |
| FDD-API-03 | `docs/FDD.md` | Contrato HTTP | `PATCH /api/v1/webhooks/:id` | TRANSCRICAO | `[09:33] Bruno` |
| FDD-API-04 | `docs/FDD.md` | Contrato HTTP | `DELETE /api/v1/webhooks/:id` | TRANSCRICAO | `[09:33] Bruno` |
| FDD-API-05 | `docs/FDD.md` | Contrato HTTP | Endpoint de rotação de secret | TRANSCRICAO | `[09:21] Sofia` |
| FDD-API-06 | `docs/FDD.md` | Contrato HTTP | Histórico de deliveries | TRANSCRICAO | `[09:34] Marcos` |
| FDD-API-07 | `docs/FDD.md` | Contrato HTTP | Replay administrativo da DLQ | TRANSCRICAO | `[09:18] Diego`; `[09:36] Sofia` |
| FDD-OUT-01 | `docs/FDD.md` | Contrato Outbound | Campos do payload e ausência de items | TRANSCRICAO | `[09:43] Diego` |
| FDD-OUT-02 | `docs/FDD.md` | Contrato Outbound | Headers do POST | TRANSCRICAO | `[09:44] Diego`; `[09:44] Sofia` |
| FDD-OUT-03 | `docs/FDD.md` | Segurança | HMAC-SHA256 sobre o corpo | TRANSCRICAO | `[09:19] Sofia`; `[09:20] Sofia` |
| FDD-OUT-04 | `docs/FDD.md` | Segurança | HTTPS obrigatório | TRANSCRICAO | `[09:23] Sofia` |
| FDD-OUT-05 | `docs/FDD.md` | Restrição | Payload maior que 64 KiB gera erro | TRANSCRICAO | `[09:23] Sofia`; `[09:24] Diego`; `[09:24] Larissa` |
| FDD-OBS-01 | `docs/FDD.md` | Observabilidade | Reusar Pino e auditar replay | TRANSCRICAO | `[09:29] Bruno`; `[09:36] Sofia` |
| FDD-OBS-02 | `docs/FDD.md` | Métrica | Medir latência fim a fim contra limite de 10s | TRANSCRICAO | `[09:02] Marcos`; `[09:10] Larissa` |
| FDD-OBS-03 | `docs/FDD.md` | Tracing | `event_id` correlaciona produção e entrega | TRANSCRICAO | `[09:25] Diego`; `[09:44] Diego` |
| ADR-001 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Outbox transacional no MySQL | TRANSCRICAO | `[09:06] Diego`; `[09:07] Diego`; `[09:40] Bruno` |
| ADR-002 | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Decisão | Worker separado com polling de 2s | TRANSCRICAO | `[09:09] Diego`; `[09:10] Larissa`; `[09:11] Diego` |
| ADR-003 | `docs/adrs/ADR-003-retry-backoff-e-dlq.md` | Decisão | Retry com backoff e DLQ | TRANSCRICAO | `[09:15] Diego`; `[09:17] Larissa`; `[09:18] Diego` |
| ADR-004 | `docs/adrs/ADR-004-hmac-sha256-por-endpoint.md` | Decisão | HMAC-SHA256 e secret por endpoint | TRANSCRICAO | `[09:20] Sofia`; `[09:21] Sofia`; `[09:22] Sofia` |
| ADR-005 | `docs/adrs/ADR-005-entrega-at-least-once.md` | Decisão | At-least-once com `X-Event-Id` | TRANSCRICAO | `[09:24] Diego`; `[09:25] Diego`; `[09:26] Larissa` |
| ADR-006 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Decisão | Reuso dos padrões do projeto | TRANSCRICAO | `[09:27] Bruno`; `[09:29] Bruno`; `[09:30] Larissa` |
| ADR-007 | `docs/adrs/ADR-007-snapshot-do-payload.md` | Decisão | Snapshot renderizado na inserção | TRANSCRICAO | `[09:51] Bruno`; `[09:52] Larissa`; `[09:52] Diego` |
| CODE-01 | `docs/FDD.md` | Integração com Código | Transação e mudança de status | CODIGO | `src/modules/orders/order.service.ts` |
| CODE-02 | `docs/FDD.md` | Integração com Código | Máquina de estados do pedido | CODIGO | `src/modules/orders/order.status.ts` |
| CODE-03 | `docs/FDD.md` | Integração com Código | Classe base e forma dos erros | CODIGO | `src/shared/errors/app-error.ts` |
| CODE-04 | `docs/FDD.md` | Integração com Código | Erros HTTP especializados | CODIGO | `src/shared/errors/http-errors.ts` |
| CODE-05 | `docs/FDD.md` | Integração com Código | Serialização central de erros | CODIGO | `src/middlewares/error.middleware.ts` |
| CODE-06 | `docs/FDD.md` | Integração com Código | Autenticação e `requireRole` | CODIGO | `src/middlewares/auth.middleware.ts` |
| CODE-07 | `docs/FDD.md` | Integração com Código | Validação Zod | CODIGO | `src/middlewares/validate.middleware.ts` |
| CODE-08 | `docs/FDD.md` | Integração com Código | Logger Pino e redaction | CODIGO | `src/shared/logger/index.ts` |
| CODE-09 | `docs/FDD.md` | Integração com Código | Composição e montagem de controllers | CODIGO | `src/app.ts` |
| CODE-10 | `docs/FDD.md` | Integração com Código | Prefixo `/api/v1` e registro de rotas | CODIGO | `src/routes/index.ts` |
| CODE-11 | `docs/FDD.md` | Integração com Código | Bootstrap e shutdown do processo | CODIGO | `src/server.ts` |
| CODE-12 | `docs/FDD.md` | Integração com Código | Fábrica do `PrismaClient` | CODIGO | `src/config/database.ts` |
| CODE-13 | `docs/FDD.md` | Integração com Código | Convenções MySQL, UUID e relações | CODIGO | `prisma/schema.prisma` |
| CODE-14 | `docs/FDD.md` | Integração com Código | Envelope de paginação | CODIGO | `src/shared/http/response.ts` |
| CODE-15 | `docs/PRD.md` | Teste/Regressão | Padrão de testes transacionais de pedidos | CODIGO | `tests/orders.test.ts` |

## Cobertura

- Os requisitos funcionais e não funcionais identificados no PRD estão
  individualmente mapeados.
- Alternativas e questões em aberto do RFC estão individualmente mapeadas.
- Fluxos, contratos, restrições, observabilidade e integrações centrais do FDD
  estão mapeados.
- Os sete ADRs têm linha própria.
- Critérios de aceite e riscos que apenas reafirmam itens anteriores herdam os
  IDs correspondentes, evitando duplicar a mesma fonte.
