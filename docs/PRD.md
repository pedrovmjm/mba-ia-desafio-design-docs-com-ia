# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Status** | Aprovado para planejamento |
| **Data de elaboração** | 2026-07-30 |
| **Responsável de produto na reunião** | Marcos |
| **Stakeholders** | Larissa, Bruno, Diego e Sofia |
| **Relacionados** | [RFC](./RFC.md) · [FDD](./FDD.md) · [ADRs](./adrs/README.md) · [Tracker](./TRACKER.md) |

## 1. Resumo e contexto

O OMS passará a enviar webhooks outbound quando o status de um pedido mudar.
Clientes B2B poderão cadastrar endpoints HTTPS, escolher os status de interesse
e receber eventos assinados em tempo quase real, sem consultar repetidamente a
API de pedidos. A feature inclui configuração, histórico, segurança, reentrega e
recuperação administrativa de falhas.

## 2. Problema e motivação

Atlas Comercial, MaxDistribuição e Nova Cargo pediram formalmente notificações
de mudança de status. Hoje seus sistemas fazem polling em `GET /orders`, o que
torna a integração lenta e cara. Para esses clientes, uma entrega abaixo de 10
segundos já é percebida como tempo real (`[09:00]`–`[09:02]`).

Existe também risco comercial: a Atlas indicou que pode migrar para um
concorrente se a necessidade não for atendida até o fim do trimestre. O pedido
de prazo foi para o fim de novembro, e o time estimou três sprints incluindo
revisão de segurança (`[09:45]`–`[09:47]`).

## 3. Público-alvo e cenários de uso

### Público-alvo

- times de integração dos clientes B2B, inicialmente Atlas, MaxDistribuição e
  Nova Cargo;
- usuários autenticados do OMS que configuram endpoints em nome de um cliente;
- administradores internos que investigam entregas e reprocessam a DLQ;
- suporte e segurança, que precisam de histórico e trilha de auditoria.

### Cenários

1. A Atlas cadastra um endpoint para `SHIPPED` e `DELIVERED` e substitui polling
   por eventos.
2. A MaxDistribuição recebe somente os status selecionados, sem ruído de outras
   transições.
3. A Nova Cargo consulta as últimas entregas para diagnosticar uma resposta
   `503`.
4. Um endpoint fica indisponível durante manutenção; a plataforma retenta e
   entrega quando ele volta.
5. Uma secret suspeita de vazamento é rotacionada sem corte imediato.
6. Depois do esgotamento dos retries, um administrador corrige a causa e
   reprocessa o item da DLQ.

## 4. Objetivos e métricas de sucesso

| ID | Objetivo | Métrica | Meta |
| --- | --- | --- | --- |
| PRD-OBJ-01 | Notificar em tempo quase real | Latência do commit à primeira entrega bem-sucedida | **p95 < 10 segundos** em condições normais |
| PRD-OBJ-02 | Evitar perda silenciosa | Eventos commitados com assinatura ativa que terminam como entregues ou DLQ | **100%** |
| PRD-OBJ-03 | Reduzir polling dos solicitantes | Clientes solicitantes que substituem o acompanhamento periódico por webhook | **3 de 3** após a adoção |
| PRD-OBJ-04 | Cumprir o compromisso comercial | Prazo do pacote pronto para produção | **3 sprints**, incluindo revisão de segurança |
| PRD-OBJ-05 | Garantir revisão de segurança | Tempo reservado antes do deploy | **mínimo de 2 dias úteis** |

## 5. Escopo

### Incluído

- webhooks somente de saída para mudanças de status de pedido;
- cadastro, edição, remoção e listagem de endpoints por cliente;
- escolha dos status recebidos por endpoint;
- geração e rotação de secret;
- entrega HTTPS assinada com HMAC-SHA256;
- histórico das últimas entregas com sucesso/falha, payload, resposta e duração;
- retry automático, DLQ persistida e replay administrativo auditado;
- garantia at-least-once e chave de deduplicação;
- integração transacional com a mudança de status.

### Fora de escopo

| Item | Motivo/origem |
| --- | --- |
| E-mail ao cliente depois de falhas repetidas | Adiado para uma fase posterior à medição de impacto (`[09:37]`–`[09:38]`) |
| Dashboard visual | Projeto separado do frontend; esta fase entrega somente API (`[09:39]`–`[09:40]`) |
| Rate limiting de saída | Observar antes de decidir (`[09:38]`–`[09:39]`) |
| Arquivamento automático de eventos entregues | Política aproximada de 30 dias foi mencionada, mas explicitamente excluída (`[09:08]`) |
| Múltiplos workers e escala horizontal | Particionamento/locking adiados (`[09:12]`–`[09:13]`) |
| Ordering global | Clientes não solicitaram essa garantia (`[09:13]`–`[09:14]`) |
| Webhooks inbound | Escopo é somente OMS → cliente (`[09:02]`–`[09:03]`) |
| Itens do pedido no evento | Cliente consulta detalhes pela API para manter o payload enxuto (`[09:43]`–`[09:44]`) |

## 6. Requisitos funcionais

| ID | Requisito |
| --- | --- |
| PRD-RF-01 | Permitir cadastrar endpoint com `customerId`, URL HTTPS e lista de status; gerar e devolver a secret na criação. |
| PRD-RF-02 | Permitir editar URL, lista de status e estado ativo do endpoint. |
| PRD-RF-03 | Permitir remover um endpoint e listar os endpoints de um cliente. |
| PRD-RF-04 | Aplicar o filtro de status antes de criar eventos: sem endpoint inscrito, não criar linha de outbox. |
| PRD-RF-05 | Quando uma mudança de status for confirmada, criar um snapshot do evento na mesma transação. |
| PRD-RF-06 | Enviar o evento por POST com `event_id`, tipo, horário, pedido, transição, cliente e total, sem itens. |
| PRD-RF-07 | Enviar `X-Event-Id`, `X-Webhook-Id`, `X-Timestamp`, `X-Signature` e `Content-Type`. |
| PRD-RF-08 | Permitir rotacionar a secret, mantendo a anterior válida por 24 horas. |
| PRD-RF-09 | Disponibilizar histórico das últimas entregas por endpoint, com sucesso/falha, payload, resposta e duração. |
| PRD-RF-10 | Retentar falhas cinco vezes nos intervalos de 1m, 5m, 30m, 2h e 12h e mover a falha final para DLQ. |
| PRD-RF-11 | Permitir que somente `ADMIN` reprocese a DLQ e registrar quem executou a ação. |
| PRD-RF-12 | Permitir CRUD de configuração a qualquer role autenticada nesta fase, recebendo `customerId` no contrato. |

## 7. Requisitos não funcionais

| ID | Requisito |
| --- | --- |
| PRD-RNF-01 | Entrega normal em menos de 10 segundos; polling do worker a cada 2 segundos. |
| PRD-RNF-02 | Chamada ao cliente não bloqueia a mudança de status; worker roda em processo separado. |
| PRD-RNF-03 | Evento e status do pedido têm consistência atômica; falha na outbox reverte a transação. |
| PRD-RNF-04 | Entrega at-least-once; duplicatas são deduplicáveis pelo `X-Event-Id`. |
| PRD-RNF-05 | Assinatura HMAC-SHA256 com secret exclusiva por endpoint e TLS obrigatório. |
| PRD-RNF-06 | Timeout de 10 segundos por tentativa. |
| PRD-RNF-07 | Payload limitado a 64 KiB; excesso causa erro, nunca truncamento. |
| PRD-RNF-08 | Ordenação por pedido enquanto houver single-worker; sem promessa de ordering global. |
| PRD-RNF-09 | Reusar os padrões do OMS: Prisma/MySQL, módulos, Zod, `AppError`, JWT e Pino. |

## 8. Decisões e trade-offs principais

- **Outbox MySQL:** atomicidade sem infraestrutura nova, ao custo de mais escritas
  e polling ([ADR-001](./adrs/ADR-001-outbox-no-mysql.md)).
- **Worker separado:** isolamento da API, ao custo de um processo adicional para
  operar ([ADR-002](./adrs/ADR-002-worker-separado-com-polling.md)).
- **Backoff limitado e DLQ:** cobre indisponibilidade longa sem retry eterno, ao
  custo de recuperação administrativa da falha final
  ([ADR-003](./adrs/ADR-003-retry-backoff-e-dlq.md)).
- **HMAC por endpoint:** reduz o raio de vazamento, mas exige gestão e rotação de
  secrets ([ADR-004](./adrs/ADR-004-hmac-sha256-por-endpoint.md)).
- **At-least-once:** evita perda em falhas incertas, transferindo ao cliente a
  responsabilidade de deduplicar
  ([ADR-005](./adrs/ADR-005-entrega-at-least-once.md)).
- **Snapshot:** preserva o estado da transição, com duplicação controlada de
  dados ([ADR-007](./adrs/ADR-007-snapshot-do-payload.md)).

## 9. Dependências

- extensão transacional de `OrderService.changeStatus`;
- novas tabelas no MySQL e migration Prisma;
- novo processo de worker usando a mesma conexão lógica de banco;
- JWT e roles já existentes para configuração/replay;
- documentação no portal de desenvolvedor sobre assinatura e idempotência;
- pelo menos dois dias úteis de revisão de Sofia antes do deploy;
- implantação em três sprints, mantendo os itens excluídos fora da primeira fase.

## 10. Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Evento ser gravado fora da transação | Média | Alto | Publisher recebe a transação corrente; revisão e teste explícito de rollback |
| Worker parar e acumular backlog | Média | Alto | Processo supervisionado; métrica de profundidade e idade da outbox |
| Secret vazar ou assinatura ser implementada incorretamente | Baixa | Alto | Secret por endpoint, rotação, redaction e revisão dedicada de Sofia |
| Cliente processar uma duplicata | Média | Médio | Documentar at-least-once e deduplicação obrigatória por `X-Event-Id` |
| Tabelas crescerem e degradarem polling | Média | Médio | Índices e lote pequeno; política de retenção será decisão futura |
| Atraso comprometer a Atlas | Média | Alto | Escopo fechado em três sprints; dashboard, e-mail, rate limiting e escala adiados |
| Um endpoint lento reduzir vazão do worker único | Média | Médio | Timeout de 10s e monitoramento; paralelismo particionado fica para evolução |

## 11. Critérios de aceitação

- [ ] Usuário autenticado cadastra, edita, lista e remove endpoints.
- [ ] Secret aparece somente na criação e rotação.
- [ ] URL sem HTTPS é rejeitada.
- [ ] Mudança para status inscrito gera evento; status não inscrito não gera.
- [ ] Endpoint saudável recebe o evento assinado em menos de 10 segundos.
- [ ] Corpo contém os campos acordados e não contém itens.
- [ ] Reenvios preservam corpo e `X-Event-Id`.
- [ ] Falha segue os cinco intervalos e termina na DLQ.
- [ ] Replay exige `ADMIN` e registra o responsável.
- [ ] Rotação mantém as duas secrets válidas por 24 horas.
- [ ] Histórico mostra tentativas com payload, resposta e duração.
- [ ] Payload acima de 64 KiB falha sem truncamento.
- [ ] Mudança de status e outbox revertem juntas em erro.
- [ ] E-mail, dashboard, rate limiting e múltiplos workers não entram na entrega.

## 12. Estratégia de testes e validação

- **Produto/E2E:** cadastro → mudança de status → recebimento assinado em menos de
  10 segundos; validar os três clientes-piloto.
- **Unidade:** filtro de status, snapshot, limite de 64 KiB, HMAC, grace period,
  backoff e códigos `WEBHOOK_*`.
- **Integração:** commit/rollback do pedido e outbox; CRUD; deliveries; replay
  permitido/negado por role.
- **Worker:** `2xx`, não-`2xx`, timeout, erro de rede, cinco retries, DLQ,
  ordering por pedido e preservação do ID.
- **Segurança:** URL HTTPS, comparação de assinatura, redaction e rotação; revisão
  final de Sofia com janela mínima de dois dias úteis.
- **Regressão:** manter verdes `tests/auth.test.ts` e `tests/orders.test.ts`.
- **Observabilidade:** simular worker parado, backlog, retry e DLQ e confirmar
  logs, métricas deriváveis e correlação por `event_id`.
