# FDD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Status** | Pronto para implementação após aprovação do RFC |
| **Data de elaboração** | 2026-07-30 |
| **Base arquitetural** | [RFC](./RFC.md) e [ADRs](./adrs/README.md) |
| **Rastreabilidade** | [Tracker](./TRACKER.md) |

## 1. Contexto e motivação técnica

O OMS altera o status de pedidos dentro de uma transação Prisma que também
registra histórico e ajusta estoque. A feature deve acrescentar notificação
outbound sem colocar uma chamada HTTP na transação nem permitir o cenário
“pedido alterado, evento perdido”. A arquitetura aprovada combina outbox
transacional, worker separado, entrega assinada, retry e DLQ.

Este documento detalha como implementar essa arquitetura. Motivações de produto
ficam no [PRD](./PRD.md); alternativas e questões em aberto, no
[RFC](./RFC.md); razões de cada decisão, nos [ADRs](./adrs/README.md).

## 2. Objetivos técnicos

| ID | Objetivo |
| --- | --- |
| FDD-OBJ-01 | Gravar a mudança de status e os eventos correspondentes atomicamente. |
| FDD-OBJ-02 | Entregar o primeiro envio em menos de 10 segundos em condições normais, com polling a cada 2 segundos. |
| FDD-OBJ-03 | Isolar API e entrega em processos distintos, usando a mesma stack e banco. |
| FDD-OBJ-04 | Autenticar o corpo com HMAC-SHA256 e secret exclusiva por endpoint. |
| FDD-OBJ-05 | Fornecer at-least-once, cinco retentativas e recuperação por DLQ. |
| FDD-OBJ-06 | Reusar módulos, Prisma, Zod, `AppError`, JWT, Pino e o formato de resposta existentes. |

## 3. Escopo e exclusões

### Incluído

- módulo `src/modules/webhooks/`;
- quatro modelos persistentes: endpoint, outbox, delivery e dead letter;
- CRUD, rotação de secret, histórico de entregas e replay de DLQ;
- publisher invocado dentro de `OrderService.changeStatus`;
- worker Node com polling, HMAC, timeout, retry e DLQ;
- métricas deriváveis, logs estruturados e correlação fim a fim.

### Excluído

- e-mail de alerta por falhas;
- dashboard visual;
- rate limiting de saída;
- arquivamento/expurgo automático;
- webhooks inbound;
- mais de um worker, particionamento e ordering global;
- itens do pedido no payload.

## 4. Desenho de componentes e dados

### 4.1 Organização proposta

```text
src/
├── worker.ts
└── modules/webhooks/
    ├── webhook.controller.ts
    ├── webhook.errors.ts
    ├── webhook.processor.ts
    ├── webhook.publisher.ts
    ├── webhook.repository.ts
    ├── webhook.routes.ts
    ├── webhook.schemas.ts
    └── webhook.service.ts
```

`webhook.publisher.ts` é a única peça chamada pelo domínio de pedidos.
`webhook.processor.ts` não conhece Express; recebe dependências e processa os
eventos. Controller e service atendem somente a API de administração/configuração.

### 4.2 Modelo persistente proposto

Todos os IDs seguem o padrão atual: UUID em `String @db.Char(36)`.

| Modelo/tabela | Campos essenciais | Índices/relações |
| --- | --- | --- |
| `WebhookEndpoint` / `webhook_endpoints` | `id`, `customerId`, `url`, `secret`, `previousSecret?`, `previousSecretExpiresAt?`, `events` (JSON de `OrderStatus[]`), `active`, timestamps | FK para `Customer`; índices em `customerId` e `(customerId, active)` |
| `WebhookOutbox` / `webhook_outbox` | `id` (= `event_id`), `webhookId`, `orderId`, `eventType`, `payload` JSON, `status`, `retryCount`, `nextAttemptAt?`, `lastError?`, timestamps | FKs para endpoint/pedido; índices em `(status, nextAttemptAt, createdAt)`, `createdAt` e `(webhookId, orderId, createdAt)` |
| `WebhookDelivery` / `webhook_deliveries` | `id`, `webhookId`, `eventId`, `attempt`, `success`, `httpStatus?`, `requestPayload`, `responseBody?`, `durationMs`, `attemptedAt` | índice em `(webhookId, attemptedAt)` e `eventId` |
| `WebhookDeadLetter` / `webhook_dead_letter` | `id`, `originalEventId`, `webhookId`, `orderId`, `payload`, `failureReason`, `failedAt`, `replayedAt?`, `replayedById?` | índices em `failedAt`, `webhookId`; FK opcional para `User` no replay |

O status da outbox usa `PENDING`, `PROCESSING`, `FAILED` e `DELIVERED`.
`FAILED` com `nextAttemptAt` representa uma falha transitória; na falha final, o
registro é movido de forma transacional para a DLQ.

Cada endpoint elegível recebe sua própria linha de outbox. Isso permite retry,
histórico, assinatura e DLQ independentes sem que um cliente/endpoint bloqueie
os demais.

### 4.3 Configuração

O worker usa a mesma `DATABASE_URL` da API e uma instância própria de
`PrismaClient`. Os parâmetros decididos são constantes/configurações validadas:

| Parâmetro | Valor |
| --- | --- |
| Intervalo de polling | `2_000 ms` |
| Timeout HTTP | `10_000 ms` |
| Backoff | `[60_000, 300_000, 1_800_000, 7_200_000, 43_200_000] ms` |
| Máximo de retentativas | `5` após o envio inicial |
| Tamanho máximo do corpo | `65_536 bytes` em UTF-8 |

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox

1. `OrderService.changeStatus` valida a transição e executa os ajustes atuais.
2. Depois de atualizar `orders` e criar `order_status_history`, mas ainda dentro
   do `$transaction`, chama
   `publishWebhookEvent(tx, refreshedOrder, fromStatus, toStatus)`.
3. O publisher consulta endpoints ativos do `customerId` cujo `events` contém
   `toStatus`. Se não houver inscritos, encerra sem escrever.
4. Para cada endpoint, gera um UUID, monta o snapshot descrito na seção 7 e
   serializa o JSON uma única vez.
5. Se os bytes UTF-8 excederem 64 KiB, lança
   `WEBHOOK_PAYLOAD_TOO_LARGE`; a transação toda reverte, sem truncamento.
6. Insere a linha `PENDING`, `retryCount = 0`, com o snapshot.
7. Qualquer erro propaga para o fluxo atual de `changeStatus`; não existe fallback
   que grave o evento depois do commit.

### 5.2 Seleção e processamento pelo worker

1. A cada 2 segundos, o worker busca um lote pequeno de:
   - `PENDING`; ou
   - `FAILED` cujo `nextAttemptAt <= now()`.
2. Ordena por `createdAt` ascendente. Um candidato só é elegível se não existir
   evento anterior ainda não terminal para o mesmo `(webhookId, orderId)`. Essa
   guarda preserva a sequência por pedido e endpoint durante retries.
3. Como há um único worker, marca o evento como `PROCESSING` e o processa
   sequencialmente.
4. Carrega o endpoint. Se estiver removido/inativo, registra falha e aplica o
   mesmo ciclo de retry/DLQ; o snapshot não é descartado silenciosamente.
5. Usa exatamente o JSON serializado do snapshot para calcular a assinatura e
   enviar `POST` com timeout de 10 segundos.
6. Registra toda tentativa em `webhook_deliveries`, incluindo status HTTP quando
   houver, corpo de resposta e duração.
7. Qualquer `2xx` é sucesso: marca `DELIVERED`. Resposta não-`2xx`, timeout ou
   falha de conexão segue para retry.

### 5.3 Retry e backoff

O envio inicial tem `attempt = 1` e `retryCount = 0`. Depois de uma falha:

| Falha ocorrida | Próxima ação |
| --- | --- |
| Envio inicial | retry 1 em 1 minuto |
| Retry 1 | retry 2 em 5 minutos |
| Retry 2 | retry 3 em 30 minutos |
| Retry 3 | retry 4 em 2 horas |
| Retry 4 | retry 5 em 12 horas |
| Retry 5 | mover para DLQ |

Em cada falha não final, o worker incrementa `retryCount`, grava `lastError`,
calcula `nextAttemptAt` e muda para `FAILED`. O envio seguinte preserva
`event_id` e corpo byte a byte. A soma dos intervalos é de aproximadamente
14h36 entre a primeira falha e a última retentativa.

### 5.4 DLQ e replay

1. Depois da quinta retentativa falhar, uma transação cria
   `webhook_dead_letter` com snapshot, IDs, último motivo e horário e remove a
   linha da outbox.
2. `POST /api/v1/admin/webhooks/dead-letter/:id/replay` exige JWT e
   `requireRole('ADMIN')`.
3. O service valida a existência e se o item ainda não foi reprocessado.
4. Na mesma transação, cria nova linha `PENDING`, com novo ID de outbox e
   `retryCount = 0`, e preenche `replayedAt`/`replayedById` na DLQ.
5. O log `webhook_dlq_replayed` registra dead letter, evento original, novo
   evento e usuário. Repetir o mesmo replay retorna conflito.

O novo ID distingue o ciclo manual do evento original; a relação fica preservada
na DLQ para auditoria.

### 5.5 Rotação de secret

1. O endpoint de rotação gera e devolve uma nova secret uma única vez.
2. A secret vigente passa a `previousSecret`, com expiração em 24 horas; a nova
   torna-se `secret`.
3. Durante o grace period, `X-Signature` contém duas assinaturas
   `sha256=<nova>,sha256=<anterior>` sobre o mesmo corpo. O consumidor aceita se
   qualquer valor verificar.
4. Após 24 horas, o worker envia somente a assinatura da secret nova e a
   anterior pode ser eliminada.

Essa representação implementa a decisão de duas secrets válidas em paralelo sem
adicionar outro header público.

## 6. Contratos públicos da API

Todos os endpoints usam `Authorization: Bearer <JWT>`, JSON e a forma de erro já
existente:

```json
{
  "error": {
    "code": "WEBHOOK_INVALID_URL",
    "message": "Webhook URL must use HTTPS",
    "details": { "url": "http://example.test/hook" }
  }
}
```

### 6.1 `POST /api/v1/webhooks`

Cadastra um endpoint. Qualquer role autenticada. A secret aparece somente nesta
resposta e em respostas de rotação.

Request:

```json
{
  "customerId": "76878cf4-bf61-44fc-9f95-4147c44457e7",
  "url": "https://atlas.example.com/oms/events",
  "events": ["SHIPPED", "DELIVERED"]
}
```

Response `201 Created`:

```json
{
  "id": "c545a4bc-fc07-49a9-b22d-8e8e5c193cba",
  "customerId": "76878cf4-bf61-44fc-9f95-4147c44457e7",
  "url": "https://atlas.example.com/oms/events",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_generated_value",
  "createdAt": "2026-07-30T18:00:00.000Z"
}
```

Status: `201`; `400 WEBHOOK_INVALID_URL`; `400 WEBHOOK_INVALID_EVENTS`;
`404 WEBHOOK_CUSTOMER_NOT_FOUND`.

### 6.2 `GET /api/v1/webhooks?customerId=<uuid>&page=1&pageSize=20`

Lista endpoints de um cliente, sem secrets.

Request:

```http
GET /api/v1/webhooks?customerId=76878cf4-bf61-44fc-9f95-4147c44457e7&page=1&pageSize=20 HTTP/1.1
Authorization: Bearer <JWT>
```

Response `200 OK`:

```json
{
  "data": [
    {
      "id": "c545a4bc-fc07-49a9-b22d-8e8e5c193cba",
      "customerId": "76878cf4-bf61-44fc-9f95-4147c44457e7",
      "url": "https://atlas.example.com/oms/events",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

Status: `200`; `400 WEBHOOK_CUSTOMER_REQUIRED`.

### 6.3 `PATCH /api/v1/webhooks/:id`

Edita URL, eventos e estado. `customerId` e secret não são alteráveis aqui.

Request:

```json
{
  "url": "https://atlas.example.com/oms/v2/events",
  "events": ["DELIVERED"],
  "active": true
}
```

Response `200 OK`:

```json
{
  "id": "c545a4bc-fc07-49a9-b22d-8e8e5c193cba",
  "customerId": "76878cf4-bf61-44fc-9f95-4147c44457e7",
  "url": "https://atlas.example.com/oms/v2/events",
  "events": ["DELIVERED"],
  "active": true,
  "updatedAt": "2026-07-30T18:15:00.000Z"
}
```

Status: `200`; `400 WEBHOOK_INVALID_URL`; `400 WEBHOOK_INVALID_EVENTS`;
`404 WEBHOOK_NOT_FOUND`.

### 6.4 `DELETE /api/v1/webhooks/:id`

Remove a configuração. Não apaga histórico nem eventos já persistidos.

Request:

```http
DELETE /api/v1/webhooks/c545a4bc-fc07-49a9-b22d-8e8e5c193cba HTTP/1.1
Authorization: Bearer <JWT>
```

Response:

```http
HTTP/1.1 204 No Content
```

Status: `204`; `404 WEBHOOK_NOT_FOUND`.

### 6.5 `POST /api/v1/webhooks/:id/rotate-secret`

Request sem corpo:

```http
POST /api/v1/webhooks/c545a4bc-fc07-49a9-b22d-8e8e5c193cba/rotate-secret HTTP/1.1
Authorization: Bearer <JWT>
```

Response `200 OK`:

```json
{
  "id": "c545a4bc-fc07-49a9-b22d-8e8e5c193cba",
  "secret": "whsec_new_generated_value",
  "previousSecretExpiresAt": "2026-07-31T18:30:00.000Z"
}
```

Status: `200`; `404 WEBHOOK_NOT_FOUND`; `409 WEBHOOK_SECRET_ROTATION_IN_PROGRESS`.

### 6.6 `GET /api/v1/webhooks/:id/deliveries?page=1&pageSize=100`

Retorna as últimas entregas em ordem decrescente, incluindo sucesso/falha,
payload, resposta e duração.

Request:

```http
GET /api/v1/webhooks/c545a4bc-fc07-49a9-b22d-8e8e5c193cba/deliveries?page=1&pageSize=100 HTTP/1.1
Authorization: Bearer <JWT>
```

Response `200 OK`:

```json
{
  "data": [
    {
      "id": "438d70b3-e80c-47df-91b7-ddbf7cf4ec87",
      "eventId": "71e24623-08b2-4a49-a4ca-0fda8a792c62",
      "attempt": 2,
      "success": false,
      "httpStatus": 503,
      "requestPayload": {
        "event_type": "order.status_changed",
        "to_status": "SHIPPED"
      },
      "responseBody": "temporarily unavailable",
      "durationMs": 342,
      "attemptedAt": "2026-07-30T18:20:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 1, "totalPages": 1 }
}
```

Status: `200`; `404 WEBHOOK_NOT_FOUND`.

### 6.7 `POST /api/v1/admin/webhooks/dead-letter/:id/replay`

Exige role `ADMIN`. Request sem corpo:

```http
POST /api/v1/admin/webhooks/dead-letter/b4314dab-d17f-4215-a8f4-aacfc39e6150/replay HTTP/1.1
Authorization: Bearer <ADMIN_JWT>
```

Response `202 Accepted`:

```json
{
  "deadLetterId": "b4314dab-d17f-4215-a8f4-aacfc39e6150",
  "originalEventId": "71e24623-08b2-4a49-a4ca-0fda8a792c62",
  "newEventId": "430b571e-ac5d-44be-b0c1-88ade5d729ed",
  "status": "PENDING"
}
```

Status: `202`; `403 FORBIDDEN`; `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`;
`409 WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED`.

## 7. Contrato de entrega para o cliente

Request enviado pelo worker:

```http
POST /oms/events HTTP/1.1
Host: atlas.example.com
Content-Type: application/json
X-Event-Id: 71e24623-08b2-4a49-a4ca-0fda8a792c62
X-Webhook-Id: c545a4bc-fc07-49a9-b22d-8e8e5c193cba
X-Timestamp: 2026-07-30T18:00:02.120Z
X-Signature: sha256=4d6c...
```

```json
{
  "event_id": "71e24623-08b2-4a49-a4ca-0fda8a792c62",
  "event_type": "order.status_changed",
  "timestamp": "2026-07-30T18:00:00.000Z",
  "order_id": "35b0d9fe-340c-4681-bcee-eb32d7ff5b87",
  "order_number": "ORD-000142",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "76878cf4-bf61-44fc-9f95-4147c44457e7",
  "total_cents": 15990
}
```

Semântica:

- qualquer resposta `2xx` confirma a tentativa;
- qualquer não-`2xx`, timeout de 10s ou erro de conexão causa retry;
- o cliente deve deduplicar por `X-Event-Id`;
- o cliente calcula HMAC-SHA256 sobre os bytes brutos do corpo e compara em
  tempo constante com um dos valores de `X-Signature`;
- `X-Timestamp` é o horário da tentativa; `timestamp` no corpo é o horário do
  evento e permanece estável nos retries;
- o payload nunca inclui itens. O estado completo pode ser obtido em
  `GET /api/v1/orders/:id`.

## 8. Matriz de erros

| Código | HTTP/contexto | Quando ocorre |
| --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | Endpoint inexistente |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `customerId` não existe |
| `WEBHOOK_CUSTOMER_REQUIRED` | 400 | Listagem sem `customerId` |
| `WEBHOOK_INVALID_URL` | 400 | URL ausente, inválida ou sem HTTPS |
| `WEBHOOK_INVALID_EVENTS` | 400 | Lista vazia ou valor fora de `OrderStatus` |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Snapshot supera 64 KiB; não há truncamento |
| `WEBHOOK_SECRET_ROTATION_IN_PROGRESS` | 409 | Nova rotação antes do fim do grace period atual |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | ID da DLQ inexistente |
| `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | Tentativa de repetir replay concluído |
| `WEBHOOK_DELIVERY_TIMEOUT` | interno | Endpoint não responde em 10s |
| `WEBHOOK_DELIVERY_REJECTED` | interno | Endpoint retorna não-`2xx` |

Os erros HTTP derivam de `AppError` e preservam a forma
`{ error: { code, message, details? } }` do middleware existente. Erros internos
do worker são valores estruturados no histórico/log e não respostas da API.

## 9. Estratégias de resiliência

| Mecanismo | Implementação |
| --- | --- |
| Atomicidade | Publisher recebe o `TransactionClient` de `changeStatus`; não abre transação independente |
| Isolamento | Worker separado da API e com `PrismaClient` próprio |
| Timeout | 10 segundos por POST; timeout conta como falha |
| Retry | Cinco reenvios programados, preservando corpo e `event_id` |
| Backoff | 1m, 5m, 30m, 2h e 12h |
| Fallback | DLQ persistida, replay manual ADMIN e auditado |
| Ordering | Single-worker, ordem de criação e bloqueio de evento posterior para o mesmo endpoint/pedido |
| Duplicatas | At-least-once e deduplicação do consumidor por `X-Event-Id` |

Não há fallback por e-mail nem rate limiting nesta fase.

## 10. Observabilidade

### Logs

Reusar Pino e eventos em `snake_case`:

- `webhook_event_enqueued`: `event_id`, `webhook_id`, `order_id`, `to_status`;
- `webhook_delivery_started|succeeded|failed`: IDs, tentativa, status e duração;
- `webhook_event_dead_lettered`: IDs e motivo;
- `webhook_dlq_replayed`: dead letter, IDs antigo/novo e `user_id`;
- `webhook_worker_started|stopped|poll_failed`: ciclo de vida do processo.

`secret`, `previousSecret`, `X-Signature`, autorização e corpo sensível de
credenciais devem ser redigidos. Nunca logar a secret devolvida na criação/rotação.

### Métricas

Mesmo sem adicionar uma biblioteca nesta decisão, modelos e logs devem permitir:

- `webhook_outbox_depth{status}` e idade do evento elegível mais antigo;
- `webhook_delivery_total{success,http_status}`;
- `webhook_delivery_duration_ms` p50/p95/p99;
- `webhook_end_to_end_latency_ms` do `createdAt` à primeira resposta `2xx`;
- `webhook_retry_total{retry_count}`;
- `webhook_dead_letter_total` e idade do item não reprocessado mais antigo.

O SLO de produto é p95 de latência ponta a ponta menor que 10 segundos em
condições normais.

### Tracing e correlação

O tracing lógico usa `event_id` em outbox, delivery, DLQ, logs e
`X-Event-Id`. Na requisição que muda o pedido, o log de enqueue associa o
`requestId` existente ao `event_id`. Assim é possível seguir:

```text
requestId da API → event_id → tentativas do worker → delivery ou DLQ → replay
```

Não se introduz SDK de tracing distribuído nesta fase.

## 11. Integração com o sistema existente

| Arquivo real | Integração necessária |
| --- | --- |
| [`src/modules/orders/order.service.ts`](../src/modules/orders/order.service.ts) | Invocar `publishWebhookEvent` dentro do `$transaction` de `changeStatus`, depois de obter os dados finais do pedido e antes do retorno/commit. Reusar o tipo `Prisma.TransactionClient`. |
| [`src/modules/orders/order.status.ts`](../src/modules/orders/order.status.ts) | `OrderStatus` e as transições válidas determinam os valores aceitos em `events`, `from_status` e `to_status`; o webhook não cria nova máquina de estados. |
| [`src/shared/errors/app-error.ts`](../src/shared/errors/app-error.ts) e [`src/shared/errors/http-errors.ts`](../src/shared/errors/http-errors.ts) | Implementar erros `WEBHOOK_*` como subclasses compatíveis e exportá-los pelo índice existente. |
| [`src/middlewares/error.middleware.ts`](../src/middlewares/error.middleware.ts) | Reusar a serialização central de `AppError`, Zod e Prisma, sem tratamento paralelo no módulo. |
| [`src/middlewares/auth.middleware.ts`](../src/middlewares/auth.middleware.ts) | Aplicar `authenticate` a todas as rotas e `requireRole('ADMIN')` ao replay da DLQ. |
| [`src/middlewares/validate.middleware.ts`](../src/middlewares/validate.middleware.ts) | Validar body/query/params com schemas Zod, incluindo UUID, enum de status e protocolo HTTPS. |
| [`src/shared/logger/index.ts`](../src/shared/logger/index.ts) | Reusar Pino e ampliar os caminhos de redaction para secrets e assinatura. |
| [`src/app.ts`](../src/app.ts) e [`src/routes/index.ts`](../src/routes/index.ts) | Construir controller/service/repository e montar rotas públicas/admin sob `/api/v1`. |
| [`src/server.ts`](../src/server.ts) e [`src/config/database.ts`](../src/config/database.ts) | Espelhar bootstrap e shutdown em `src/worker.ts`; criar `PrismaClient` próprio do processo. |
| [`prisma/schema.prisma`](../prisma/schema.prisma) | Adicionar enums, quatro modelos, relações e índices usando as convenções MySQL/UUID atuais. |
| [`src/shared/http/response.ts`](../src/shared/http/response.ts) | Reusar o envelope paginado nas listagens de endpoints e deliveries. |
| [`tests/orders.test.ts`](../tests/orders.test.ts) | Estender testes transacionais de mudança de status e manter as regressões atuais verdes. |

## 12. Dependências e compatibilidade

- **Runtime:** Node.js `>=20`; `fetch`, `AbortSignal` e `node:crypto` atendem
  HTTP, timeout, geração e HMAC sem pacote novo.
- **Aplicação:** Express 4, Prisma 5, Zod 3 e Pino 9 já presentes.
- **Banco:** MySQL atual, com migration Prisma para as quatro tabelas.
- **Autenticação:** JWT existente; não é criado novo tipo de credencial para a
  API de configuração.
- **Compatibilidade:** nenhuma rota atual muda. O payload outbound nasce na
  primeira versão com `event_type = "order.status_changed"`.
- **Deploy:** API e worker usam o mesmo artefato/configuração, mas comandos e
  processos distintos.

## 13. Critérios de aceite técnicos

- [ ] Mudança de status com endpoint inscrito grava pedido, histórico, estoque e
  outbox na mesma transação; rollback não deixa evento.
- [ ] Status sem endpoint inscrito não gera outbox.
- [ ] Primeiro envio saudável ocorre em menos de 10 segundos, com polling de 2s.
- [ ] Worker roda fora do processo da API e encerra o PrismaClient corretamente.
- [ ] Corpo entregue corresponde ao snapshot e não contém itens.
- [ ] Headers incluem `X-Event-Id`, `X-Webhook-Id`, `X-Timestamp`,
  `X-Signature` e `Content-Type`.
- [ ] HMAC-SHA256 valida com a secret do endpoint e rotação aceita secret nova e
  anterior durante 24 horas.
- [ ] URL HTTP é rejeitada e corpo acima de 64 KiB falha sem truncamento.
- [ ] Falha produz retries após 1m, 5m, 30m, 2h e 12h; falha final vai à DLQ.
- [ ] Retentativas preservam `event_id` e corpo byte a byte.
- [ ] Replay só funciona para `ADMIN`, é idempotente e registra quem o executou.
- [ ] Histórico expõe tentativas, payload, resposta e duração.
- [ ] Ordenação por endpoint/pedido é preservada com single-worker.
- [ ] Logs, métricas deriváveis e correlação por `event_id` cobrem o fluxo.
- [ ] Testes existentes de auth e pedidos continuam verdes.

## 14. Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Inserção fora da transação causar perda de evento | Média | Alto | Publisher recebe `tx`; teste força rollback e revisa o ponto exato em `changeStatus` |
| Worker parado violar latência | Média | Alto | Supervisão do processo, métrica de profundidade e idade do backlog |
| Vazamento de secret | Baixa | Alto | Secret por endpoint, redaction, rotação e revisão de segurança |
| Cliente processar duplicata | Média | Médio | Contrato at-least-once, ID estável e documentação de idempotência |
| Crescimento das tabelas degradar polling | Média | Médio | Índices, lote pequeno e métricas; retenção volta como decisão futura |
| Retry de evento antigo quebrar ordering | Média | Médio | Bloquear evento posterior do mesmo `(webhookId, orderId)` até estado terminal |
| Resposta lenta consumir o worker único | Média | Médio | Timeout de 10s; avaliar paralelismo particionado em fase futura |
| Regra de rotação ser implementada de forma incompatível | Baixa | Alto | Testes com duas assinaturas durante 24h e revisão de Sofia antes do deploy |
