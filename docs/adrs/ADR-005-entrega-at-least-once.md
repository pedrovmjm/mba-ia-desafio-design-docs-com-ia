# ADR-005 — Entrega at-least-once com `X-Event-Id`

- **Status:** Aceita
- **Data de registro:** 2026-07-30
- **Decisores:** Larissa, Diego, Sofia e Marcos
- **Relacionados:** [ADR-003](./ADR-003-retry-backoff-e-dlq.md), [ADR-007](./ADR-007-snapshot-do-payload.md)

## Contexto

Uma falha de rede pode ocorrer depois de o cliente processar a mensagem, mas
antes de o worker receber a resposta. O retry é necessário para não perder
eventos, porém pode produzir duplicatas. Garantir exactly-once exigiria
coordenação entre sistemas controlados por organizações diferentes.

Fontes: `[09:24]`–`[09:26]`.

## Decisão

Oferecer entrega **at-least-once**. O UUID da linha original da outbox será o
`event_id` e será enviado no corpo e no header `X-Event-Id`. Todas as
retentativas do mesmo evento preservam o mesmo ID e o mesmo snapshot.

O contrato público instruirá o cliente a tornar o consumidor idempotente,
registrando os IDs já processados e ignorando duplicatas.

## Alternativas Consideradas

### Exactly-once

Descartada porque requer protocolo e estado coordenados nos dois lados, com
complexidade desproporcional à necessidade relatada.

### At-most-once, sem retry

Descartada porque falhas transitórias causariam perda definitiva de notificações.

### Gerar novo ID a cada retry

Descartada porque impediria o cliente de reconhecer que os reenvios representam
o mesmo evento lógico.

## Consequências

### Positivas

- Falhas transitórias não causam perda silenciosa.
- UUID oferece uma chave simples e estável de deduplicação e correlação.
- A implementação não depende de estado transacional no sistema do cliente.

### Negativas

- Duplicatas são esperadas e fazem parte do contrato.
- A idempotência passa a ser responsabilidade explícita do consumidor.
- Clientes que ignorarem `X-Event-Id` podem executar efeitos duas vezes.
