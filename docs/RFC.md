# RFC — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Autor** | Pedro, com apoio do Codex |
| **Status** | Em revisão |
| **Data de elaboração** | 2026-07-30 |
| **Revisores** | Larissa (Tech Lead), Marcos (Product Manager), Bruno (Engenharia de Pedidos), Diego (Plataforma), Sofia (Segurança) |
| **Origem** | [`TRANSCRICAO.md`](../TRANSCRICAO.md) e código atual do OMS |
| **Relacionados** | [PRD](./PRD.md) · [FDD](./FDD.md) · [ADRs](./adrs/README.md) · [Tracker](./TRACKER.md) |

## TL;DR

Propõe-se um sistema de webhooks HTTP **outbound** para notificar clientes B2B
quando o status de seus pedidos mudar. O evento será gravado em uma outbox no
MySQL dentro da mesma transação que altera o pedido. Um worker Node separado
consultará a outbox a cada 2 segundos e fará entregas HTTPS assinadas com
HMAC-SHA256.

Falhas terão cinco retentativas (`1m`, `5m`, `30m`, `2h`, `12h`) e, depois,
Dead-Letter Queue (DLQ) persistida com replay administrativo. O contrato será
at-least-once: o cliente deduplicará pelo `X-Event-Id`. A proposta não adiciona
Redis, broker ou logger novo e segue os padrões atuais de módulos, Prisma, Zod,
`AppError`, autenticação e Pino. A estimativa acordada é de três sprints,
incluindo revisão de segurança.

## Contexto e problema

Atlas Comercial, MaxDistribuição e Nova Cargo pediram notificação em tempo real
sobre mudanças de status. Atualmente, esses clientes consultam `GET /orders`
periodicamente, tornando suas integrações lentas e caras. Para eles, uma entrega
em menos de 10 segundos já atende à percepção de tempo real. Há urgência
comercial: a Atlas indicou risco de migração para um concorrente se a necessidade
não for atendida no prazo (`[09:00]`–`[09:02]`).

O OMS não oferece mecanismo de eventos externos. A mudança de status já ocorre
sob transação no `OrderService.changeStatus`, envolvendo pedido, histórico e,
conforme a transição, estoque. A entrega HTTP não pode prolongar nem condicionar
essa transação. O escopo confirmado é somente **OMS → cliente**; webhooks inbound
não fazem parte da proposta.

## Proposta técnica

A solução tem quatro responsabilidades:

1. **Configuração:** módulo `webhooks` expõe API autenticada para cadastrar,
   listar, editar, remover e rotacionar endpoints, além de consultar entregas. O
   cliente informa o `customerId` e os status desejados; o replay de DLQ é
   exclusivo de `ADMIN`.
2. **Publicação atômica:** `changeStatus` cria, para cada endpoint inscrito, um
   evento com snapshot do pedido na mesma transação SQL. Se a transação reverter,
   o evento não existe; se a outbox falhar, a mudança também reverte.
3. **Entrega assíncrona:** um processo worker independente consulta eventos
   elegíveis por polling de 2 segundos, em ordem de criação, e envia POST HTTPS
   assinado. Esta fase opera com uma única instância.
4. **Recuperação:** falhas transitórias entram em backoff; após a quinta
   retentativa, o evento vai para uma tabela de DLQ e pode ser reprocessado
   manualmente por um administrador, com auditoria.

```text
PATCH pedido ── transação MySQL ──> pedido + histórico + estoque + outbox
                                                             │
                                                       polling 2s
                                                             ▼
cliente <── POST HTTPS + HMAC ── worker separado ── retry ──> DLQ
                                                              │
                                                   replay ADMIN
                                                              └──> outbox
```

Cada endpoint tem secret própria. A assinatura HMAC-SHA256 cobre exatamente o
corpo enviado; TLS é obrigatório e a secret pode ser rotacionada com convivência
de 24 horas. A entrega é at-least-once, mantendo `event_id` e snapshot entre
reenvios. O destinatário deve usar `X-Event-Id` para idempotência.

O worker, os endpoints e os modelos continuam na stack Node.js, TypeScript,
Express, Prisma e MySQL. O detalhamento de modelos, fluxos, contratos, erros e
integrações está no [FDD](./FDD.md).

## Alternativas consideradas

| Alternativa | Trade-off que levou ao descarte | Fonte |
| --- | --- | --- |
| HTTP síncrono dentro de `changeStatus` | Cliente lento ou indisponível prolongaria a transação, bloquearia mudanças e poderia provocar rollback de uma operação válida. | `[09:03]`–`[09:06]` |
| Redis Streams ou broker externo | Resolveria o desacoplamento, mas adicionaria infraestrutura e operação consideradas excessivas para o time e o volume atual. | `[09:07]` |
| Trigger MySQL para notificação reativa | Trigger só executa SQL e não acorda um processo externo; polling de 2s atende o limite de 10s sem improvisos. | `[09:09]`–`[09:10]` |
| Worker dentro do processo da API | Compartilharia ciclo de vida e recursos com o servidor HTTP; reinícios afetariam ambos. | `[09:11]` |
| Três retentativas | Encerraria o ciclo cedo demais para indisponibilidades de algumas horas já observadas. | `[09:15]`–`[09:17]` |
| Retry indefinido | Manteria eventos sem destino pendurados para sempre. | `[09:15]` |
| Exactly-once | Exigiria coordenação de estado entre OMS e cliente; at-least-once com chave de deduplicação cobre a necessidade com menor complexidade. | `[09:24]`–`[09:26]` |
| Secret global | Um vazamento comprometeria todos os clientes; secret por endpoint contém o impacto. | `[09:20]`–`[09:22]` |

## Questões em aberto

1. **Rate limiting de saída:** a primeira fase enviará uma chamada por evento,
   sem limitar a taxa por cliente. O time observará o comportamento antes de
   decidir se precisa controlar rajadas (`[09:38]`–`[09:39]`).
2. **Escala e ordering com múltiplos workers:** uma instância e a ordenação por
   criação preservam a sequência por pedido nesta fase. Particionamento por
   `orderId` ou lock pessimista ficam para uma evolução (`[09:12]`–`[09:14]`).
3. **Política de arquivamento:** a reunião cogitou arquivar entregues depois de
   aproximadamente 30 dias, mas não definiu retenção nem processo (`[09:08]`).
4. **Autorização mais granular no CRUD:** nesta fase qualquer role autenticada
   pode gerenciar configurações, com `customerId` no body/path. Endurecimento de
   permissão foi adiado (`[09:31]`–`[09:37]`).

## Impacto e riscos

- **Pedidos:** a mudança crítica está em
  [`src/modules/orders/order.service.ts`](../src/modules/orders/order.service.ts).
  A chamada ao publisher precisa continuar dentro do `$transaction`; qualquer
  deslocamento para depois do commit quebra a garantia central.
- **Banco:** entram tabelas de endpoints, outbox, entregas e DLQ. Índices da
  outbox em status e data de criação são necessários para evitar varredura
  completa à medida que o volume cresce.
- **Operação:** passa a existir um processo independente a implantar e
  monitorar. Se ele parar, os eventos não se perdem, mas o backlog viola a meta
  de latência.
- **Segurança:** o OMS passa a enviar dados de pedido para URLs externas e a
  manter secrets de assinatura. Sofia deve revisar HMAC e geração/rotação de
  secret por pelo menos dois dias úteis antes do deploy (`[09:46]`).
- **Integração do cliente:** duplicatas são possíveis. A documentação pública
  precisa destacar idempotência via `X-Event-Id`, responsabilidade assumida por
  Marcos (`[09:25]`–`[09:26]`).
- **Prazo:** a estimativa de três sprints depende da manutenção do escopo; e-mail,
  dashboard, rate limiting, arquivamento e escala horizontal continuam adiados.

## Decisões relacionadas

- [ADR-001 — Outbox transacional no MySQL](./adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Worker separado com polling de 2 segundos](./adrs/ADR-002-worker-separado-com-polling.md)
- [ADR-003 — Retry com backoff e DLQ persistida](./adrs/ADR-003-retry-backoff-e-dlq.md)
- [ADR-004 — HMAC-SHA256 com secret por endpoint](./adrs/ADR-004-hmac-sha256-por-endpoint.md)
- [ADR-005 — Entrega at-least-once com `X-Event-Id`](./adrs/ADR-005-entrega-at-least-once.md)
- [ADR-006 — Reuso dos padrões existentes](./adrs/ADR-006-reuso-dos-padroes-existentes.md)
- [ADR-007 — Snapshot do payload](./adrs/ADR-007-snapshot-do-payload.md)
