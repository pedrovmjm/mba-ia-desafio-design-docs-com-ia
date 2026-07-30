# ADR-006 — Reuso dos padrões existentes do OMS

- **Status:** Aceita
- **Data de registro:** 2026-07-30
- **Decisores:** Larissa, Bruno e Diego
- **Relacionados:** [ADR-001](./ADR-001-outbox-no-mysql.md), [ADR-002](./ADR-002-worker-separado-com-polling.md)

## Contexto

O OMS organiza cada domínio em controller, service, repository, routes e schemas.
Já possui validação Zod, erros derivados de `AppError`, tratamento centralizado,
autenticação/autorização JWT, logger Pino, cliente Prisma e UUIDs nos modelos.
Uma arquitetura paralela elevaria custo de manutenção sem benefício identificado.

Fontes: `[09:27]`–`[09:30]`, `[09:36]`, `[09:48]` e:

- [`src/modules/orders/`](../../src/modules/orders/)
- [`src/shared/errors/`](../../src/shared/errors/)
- [`src/middlewares/error.middleware.ts`](../../src/middlewares/error.middleware.ts)
- [`src/middlewares/auth.middleware.ts`](../../src/middlewares/auth.middleware.ts)
- [`src/shared/logger/index.ts`](../../src/shared/logger/index.ts)
- [`prisma/schema.prisma`](../../prisma/schema.prisma)

## Decisão

- Criar `src/modules/webhooks/` com `controller`, `service`, `repository`,
  `routes`, `schemas`, `errors`, `publisher` e `processor`.
- Usar Zod e o middleware `validate` nos contratos de entrada.
- Criar erros derivados de `AppError`, todos com prefixo `WEBHOOK_`.
- Reusar `authenticate` no CRUD e `requireRole('ADMIN')` no replay da DLQ.
- Reusar Pino, ampliando a lista de campos sensíveis redigidos para secrets.
- Manter Prisma/MySQL e IDs UUID `@db.Char(36)`.
- Registrar as rotas sob `/api/v1`, como os demais módulos.
- O worker terá seu próprio `PrismaClient`, pois executa em outro processo.

## Alternativas Consideradas

### Estrutura e camada de erros próprias

Descartada porque tornaria o módulo inconsistente com a codebase e duplicaria
tratamento já fornecido pelo middleware central.

### Novo logger, ORM ou serviço de fila

Descartada por adicionar dependências e operação sem necessidade para o escopo
decidido.

### Compartilhar a instância do `PrismaClient` da API

Impossível entre processos Node separados. Cada entry point cria sua instância,
embora use a mesma configuração e banco.

## Consequências

### Positivas

- Menor curva de aprendizado, menos dependências e comportamento uniforme.
- Autenticação, erros e logs seguem os contratos já conhecidos.
- Testabilidade e composição permanecem alinhadas com `buildApp`.

### Negativas

- Alguns arquivos existentes precisarão ser estendidos (`app.ts`, rotas, schema,
  env, logger e `order.service.ts`).
- O pool de conexões precisa considerar API e worker como processos distintos.
- O padrão atual não fornece métricas/tracing dedicados; a primeira versão usará
  dados estruturados e correlação por `event_id`.
