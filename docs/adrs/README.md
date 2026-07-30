# Architectural Decision Records

Este diretório registra as decisões arquiteturais do Sistema de Webhooks de
Notificação de Pedidos. Os documentos usam uma variante enxuta do formato MADR.

| ADR | Decisão | Status |
| --- | --- | --- |
| [ADR-001](./ADR-001-outbox-no-mysql.md) | Outbox transacional no MySQL | Aceita |
| [ADR-002](./ADR-002-worker-separado-com-polling.md) | Worker separado com polling de 2 segundos | Aceita |
| [ADR-003](./ADR-003-retry-backoff-e-dlq.md) | Cinco retentativas com backoff e DLQ | Aceita |
| [ADR-004](./ADR-004-hmac-sha256-por-endpoint.md) | HMAC-SHA256 e secret por endpoint | Aceita |
| [ADR-005](./ADR-005-entrega-at-least-once.md) | Entrega at-least-once com `X-Event-Id` | Aceita |
| [ADR-006](./ADR-006-reuso-dos-padroes-existentes.md) | Reuso dos padrões da aplicação | Aceita |
| [ADR-007](./ADR-007-snapshot-do-payload.md) | Snapshot do payload na outbox | Aceita |

As fontes primárias são a reunião em
[`TRANSCRICAO.md`](../../TRANSCRICAO.md) e o código atual. A rastreabilidade
item a item está em [`docs/TRACKER.md`](../TRACKER.md).
