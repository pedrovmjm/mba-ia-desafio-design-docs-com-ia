# ADR-004 — HMAC-SHA256 com secret por endpoint

- **Status:** Aceita
- **Data de registro:** 2026-07-30
- **Decisora principal:** Sofia
- **Relacionados:** [ADR-005](./ADR-005-entrega-at-least-once.md), [ADR-007](./ADR-007-snapshot-do-payload.md)

## Contexto

O webhook leva dados de pedido para fora da infraestrutura do OMS. O destinatário
precisa verificar a origem e a integridade do corpo. Uma credencial global
ampliaria o impacto do vazamento de um único cliente, e o time já observou um
caso de secret exposta em logs do lado de um cliente.

Fontes: `[09:19]`–`[09:24]`, `[09:44]`–`[09:45]`.

## Decisão

- Gerar uma secret exclusiva para cada endpoint e devolvê-la na criação.
- Calcular `HMAC-SHA256(secret, rawBodyUtf8)` sobre exatamente os bytes enviados.
- Enviar a assinatura em `X-Signature`, junto de `X-Event-Id`,
  `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`.
- Aceitar somente URLs `https`.
- Permitir rotação por API. A secret anterior permanece válida por 24 horas e,
  depois, deixa de ser utilizável.
- Rejeitar payload serializado acima de 64 KiB, sem truncar.
- Nunca incluir secrets em payloads de listagem, logs ou histórico de entregas.

A revisão do código de HMAC e geração/rotação de secret por Sofia deve reservar
pelo menos dois dias úteis antes do deploy.

## Alternativas Consideradas

### Secret global da plataforma

Descartada porque o vazamento de uma credencial comprometeria todos os endpoints.

### Confiar apenas em TLS

Descartada porque TLS protege o transporte, mas não oferece ao cliente uma prova
aplicacional verificável de que a mensagem foi emitida pelo OMS.

### Rotação com corte imediato

Descartada porque criaria indisponibilidade durante a troca coordenada nos
sistemas do cliente. O período de convivência de 24 horas permite migração.

## Consequências

### Positivas

- O cliente consegue autenticar a origem e detectar alteração do corpo.
- O raio de impacto de vazamento fica restrito a um endpoint.
- A rotação reduz downtime de integração.

### Negativas

- O OMS precisa manter a secret disponível para assinar, inclusive a anterior
  durante o grace period.
- Clientes precisam implementar verificação sobre o corpo bruto, sem
  reserializá-lo.
- Rotação, redação de logs e expiração adicionam estados que precisam de testes
  de segurança.
