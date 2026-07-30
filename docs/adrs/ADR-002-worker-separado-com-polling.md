# ADR-002 — Worker separado com polling de 2 segundos

- **Status:** Aceita
- **Data de registro:** 2026-07-30
- **Decisores:** Larissa, Bruno e Diego
- **Relacionados:** [ADR-001](./ADR-001-outbox-no-mysql.md), [ADR-003](./ADR-003-retry-backoff-e-dlq.md)

## Contexto

Depois de persistidos na outbox, os eventos precisam ser entregues sem ocupar o
processo HTTP da API. O MySQL não oferece um mecanismo equivalente ao
`LISTEN/NOTIFY` do PostgreSQL para acordar um consumidor externo. O requisito de
produto aceita latência inferior a 10 segundos.

Fontes: `[09:08]`–`[09:13]`, além do padrão de inicialização existente em
[`src/server.ts`](../../src/server.ts) e da fábrica Prisma em
[`src/config/database.ts`](../../src/config/database.ts).

## Decisão

Criar um entry point `src/worker.ts`, executado como processo Node independente
da API. Ele terá seu próprio `PrismaClient`, usando a mesma `DATABASE_URL`, e:

1. consultará a cada **2 segundos** um lote pequeno de eventos elegíveis;
2. ordenará os eventos mais antigos por `createdAt`;
3. processará uma única instância de worker nesta fase;
4. marcará cada resultado como entregue ou o encaminhará ao fluxo de retry.

A lógica ficará em `src/modules/webhooks/webhook.processor.ts`; o entry point
será apenas responsável por bootstrap, loop e encerramento gracioso.

## Alternativas Consideradas

### Worker dentro do processo da API

Descartada porque vincularia o ciclo de vida da entrega ao servidor HTTP. Uma
reinicialização ou falha da API interromperia o consumidor e competiria pelos
recursos do mesmo processo.

### Trigger no MySQL

Descartada porque triggers executam SQL, mas não notificam diretamente um
processo externo. A adaptação necessária seria frágil e o polling de 2 segundos
já atende a meta de menos de 10 segundos.

### Vários workers desde a primeira versão

Adiada. A paralelização pode quebrar a ordenação por pedido e exigiria
particionamento por `orderId` ou locking pessimista.

## Consequências

### Positivas

- API e consumidor têm isolamento de falha e de deploy.
- A latência de polling cabe no requisito de produto.
- Não há nova tecnologia de infraestrutura.
- O processamento mais antigo primeiro preserva a ordenação por pedido enquanto
  houver um único worker.

### Negativas

- Surge um segundo processo para implantar, supervisionar e observar.
- Um worker parado acumula backlog até voltar.
- O polling produz consultas periódicas mesmo quando a fila está vazia.
- A garantia de ordenação é limitada ao cenário de single-worker; não há
  ordenação global contratual.
