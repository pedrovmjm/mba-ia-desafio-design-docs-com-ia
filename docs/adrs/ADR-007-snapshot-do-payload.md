# ADR-007 — Snapshot do payload na criação do evento

- **Status:** Aceita
- **Data de registro:** 2026-07-30
- **Decisores:** Larissa, Bruno e Diego
- **Relacionados:** [ADR-001](./ADR-001-outbox-no-mysql.md), [ADR-004](./ADR-004-hmac-sha256-por-endpoint.md), [ADR-005](./ADR-005-entrega-at-least-once.md)

## Contexto

Entre a mudança de status e uma entrega — especialmente após retries — o pedido
pode mudar novamente. Se o worker consultar o estado atual, um evento antigo
poderia apresentar dados de uma transição posterior e deixar de representar o
fato ocorrido.

Fonte: `[09:43]`–`[09:44]`, `[09:51]`–`[09:52]`.

## Decisão

Renderizar e persistir o payload JSON completo quando o evento é inserido na
outbox, dentro da transação de `changeStatus`. O snapshot contém:

- `event_id`, `event_type` e timestamp ISO 8601;
- `order_id`, `order_number`, `from_status`, `to_status`;
- `customer_id` e `total_cents`;
- nenhum item do pedido.

O worker enviará o snapshot sem reconstruí-lo. Retentativas usam o mesmo corpo e
o mesmo `event_id`. Se o cliente precisar do estado atual ou dos itens, consultará
`GET /api/v1/orders/:id`.

## Alternativas Consideradas

### Persistir somente `order_id` e renderizar no envio

Descartada porque o estado consultado no futuro pode divergir da transição que
originou o evento.

### Incluir itens do pedido

Descartada na reunião para manter o evento enxuto. Detalhes continuam disponíveis
pela API de pedidos.

## Consequências

### Positivas

- O evento é uma representação imutável da mudança ocorrida.
- Retry é determinístico e mantém corpo/assinatura coerentes.
- O cliente recebe apenas os campos essenciais.

### Negativas

- Há duplicação de dados no MySQL.
- Alterações futuras no contrato exigirão versionamento ou compatibilidade com
  snapshots já pendentes.
- O limite de 64 KiB precisa ser verificado antes da inserção.
