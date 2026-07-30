# ADR-001 — Outbox transacional no MySQL

- **Status:** Aceita
- **Data de registro:** 2026-07-30
- **Decisores:** Larissa, Bruno e Diego
- **Relacionados:** [ADR-002](./ADR-002-worker-separado-com-polling.md), [ADR-007](./ADR-007-snapshot-do-payload.md)

## Contexto

O `OrderService.changeStatus` já executa, em uma transação Prisma, a mudança
do pedido, a atualização de estoque quando aplicável e a criação do histórico.
Fazer a chamada HTTP dentro dessa transação a deixaria dependente da latência e
da disponibilidade do cliente. Gravar o evento depois do commit, por outro lado,
criaria a possibilidade de o status mudar sem que a notificação fosse registrada.

Fontes: `[09:03]`–`[09:07]`, `[09:40]`–`[09:41]` e
[`src/modules/orders/order.service.ts`](../../src/modules/orders/order.service.ts).

## Decisão

Usar o padrão **Transactional Outbox** no MySQL existente. Para cada endpoint
ativo e inscrito no novo status, `changeStatus` chamará
`publishWebhookEvent(tx, order, fromStatus, toStatus)` com o mesmo
`Prisma.TransactionClient` da transação corrente. A função gravará uma linha na
`webhook_outbox`.

Assim:

- commit da mudança de status implica commit do evento;
- rollback da mudança remove também a criação do evento;
- falha ao publicar na outbox provoca rollback da transação inteira;
- a entrega HTTP ocorre depois, fora da requisição, por um worker.

## Alternativas Consideradas

### Chamada HTTP síncrona no `changeStatus`

Descartada porque cliente lento ou indisponível prolongaria a transação e poderia
bloquear ou reverter uma mudança de status válida.

### Redis Streams ou outra fila externa

Descartada nesta fase porque exigiria infraestrutura e operação adicionais para
um time pequeno. A outbox no banco existente fornece a atomicidade necessária sem
um novo componente distribuído.

### Gravação assíncrona depois do commit

Descartada porque existe uma janela entre o commit do pedido e a gravação do
evento na qual uma falha causaria perda silenciosa da notificação.

## Consequências

### Positivas

- Consistência atômica entre pedido, histórico, estoque e evento.
- Indisponibilidade do cliente não participa da transação de negócio.
- Reuso do MySQL e do Prisma já operados pelo time.

### Negativas

- Cada endpoint inscrito adiciona uma escrita à transação de mudança de status.
- O MySQL passa a exercer também o papel de buffer de eventos.
- Erro na gravação da outbox impede a mudança de status; esse acoplamento é
  intencional para preservar a garantia de não perder eventos.
- É necessário controlar backlog e, futuramente, definir arquivamento.
