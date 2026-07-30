# Da reunião ao documento — Webhooks de pedidos

## Sobre o desafio

Este repositório transforma a transcrição de uma reunião técnica e o código de
um Order Management System em um pacote de design docs para um Sistema de
Webhooks de Notificação de Pedidos. A entrega é exclusivamente documental:
define o problema, a proposta arquitetural, as decisões, o desenho acionável de
implementação e a origem de cada requisito, sem alterar a aplicação.

O resultado foi construído tratando [`TRANSCRICAO.md`](./TRANSCRICAO.md) e o
código atual como fontes primárias. Todos os documentos foram redigidos e
auditados contra essas fontes locais.

## Ferramentas de IA utilizadas

- **OpenAI Codex:** leitura dirigida da transcrição e do repositório, extração de
  decisões, redação dos documentos, revisão cruzada e automação das verificações.
- **Exploração local com Git, PowerShell e ripgrep:** não é IA, mas deu ao Codex
  contexto verificável sobre branches, estrutura, símbolos e caminhos reais.

## Workflow adotado

1. **Proteção do trabalho:** inspeção do `git status` e criação da branch
   `feat/desafio-design-docs`.
2. **Leitura do enunciado:** transformação da checklist de aceite em uma lista de
   verificações objetivas.
3. **Extração da reunião:** classificação de cada fala como requisito, decisão,
   alternativa descartada, questão em aberto, risco ou fora de escopo.
4. **Mapeamento do código:** leitura de pedidos, transação, máquina de estados,
   Prisma, erros, autenticação, validação, logger, composição e testes.
5. **ADRs primeiro:** registro das sete decisões para formar o esqueleto estável
   da solução.
6. **RFC e FDD:** consolidação da arquitetura no RFC e detalhamento de dados,
   fluxos, contratos e integração no FDD.
7. **PRD:** retorno ao nível de produto, removendo detalhes que pertenciam ao FDD.
8. **Tracker:** mapeamento individual de requisitos, alternativas, fluxos,
   contratos, ADRs e caminhos de código.
9. **Auditoria final:** contagem de ADRs/requisitos/endpoints, validação de links
    e caminhos, proporção das fontes e busca de contradições.

## Prompts customizados

O primeiro prompt foi usado para impedir que menções descartadas virassem
requisitos e para manter evidência explícita:

```text
Leia TRANSCRICAO.md como fonte normativa. Para cada fala relevante, produza uma
linha com: timestamp, falante, categoria (decisão fechada, requisito funcional,
requisito não funcional, alternativa descartada, questão em aberto ou fora de
escopo), resumo e documento de destino. Não transforme hipótese ou item adiado
em requisito. Se duas falas forem ambíguas, registre a ambiguidade em vez de
inventar uma decisão.
```

O segundo prompt dirigiu o mapeamento da feature sobre a aplicação real:

```text
Inspecione a codebase sem editar src/, prisma/, tests/ ou configurações. Mapeie a
feature de webhooks para caminhos e símbolos existentes: transação de
changeStatus, OrderStatus, AppError, error middleware, validate/Zod,
authenticate/requireRole, Pino, PrismaClient, buildApp e buildApiRouter. Para
cada integração, informe caminho real, responsabilidade atual, mudança proposta
e risco de regressão. Não cite arquivo que não exista.
```

O terceiro prompt foi aplicado como revisão de consistência:

```text
Audite PRD, RFC, FDD, ADRs e TRACKER como um único pacote. Verifique fronteiras:
PRD responde por quê/o quê; RFC propõe arquitetura e alternativas; ADR registra
uma decisão isolada; FDD especifica como construir. Procure divergências em
URLs, número de retries, headers, roles, limite de payload, ordering, nomes de
arquivos e itens fora de escopo. Exija uma fonte rastreável para cada requisito,
decisão ou restrição.
```

## Iterações e ajustes

Foram quatro iterações principais:

1. **Extração de requisitos:** a primeira passagem misturava decisões com ideias
   apenas discutidas. A classificação por timestamp separou e-mail, dashboard,
   rate limiting, arquivamento e múltiplos workers como itens adiados.
2. **Aderência ao código:** uma versão genérica falava em autenticação e rotas
   sem ancoragem. A leitura do repositório corrigiu o desenho para usar
   `authenticate`, `requireRole`, `AppError`, o prefixo real `/api/v1`, a
   transação de `changeStatus` e um `PrismaClient` por processo.
3. **Revisão dos contratos:** a política de retry tinha ambiguidade entre envio
   inicial e retentativas. O FDD passou a distinguir os dois conceitos, preservar
   ordering durante backoff e definir como as duas secrets funcionam no grace
   period de 24 horas.
4. **Revisão de fronteiras e rastreabilidade:** detalhes de payload/modelo foram
   mantidos no FDD, o RFC foi reduzido ao nível de proposta e o Tracker passou a
   mapear separadamente requisitos, alternativas, contratos, ADRs e arquivos.

## Como navegar a entrega

Ordem sugerida:

1. [`docs/PRD.md`](./docs/PRD.md) — problema, público, escopo, requisitos e
   sucesso.
2. [`docs/RFC.md`](./docs/RFC.md) — proposta arquitetural, alternativas e
   questões abertas.
3. [`docs/adrs/README.md`](./docs/adrs/README.md) — índice das sete decisões e
   seus trade-offs.
4. [`docs/FDD.md`](./docs/FDD.md) — fluxos, dados, endpoints, erros,
   resiliência, observabilidade e integração com o código.
5. [`docs/TRACKER.md`](./docs/TRACKER.md) — trilha de cada item até a reunião ou
   o código.
6. [`TRANSCRICAO.md`](./TRANSCRICAO.md) — fonte original para auditoria.

O enunciado original está disponível no
[repositório-base do desafio](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).
