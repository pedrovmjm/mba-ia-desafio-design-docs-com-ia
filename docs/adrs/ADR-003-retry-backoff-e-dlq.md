# ADR-003 — Retry com backoff e DLQ persistida

- **Status:** Aceita
- **Data de registro:** 2026-07-30
- **Decisores:** Larissa, Bruno e Diego
- **Relacionados:** [ADR-002](./ADR-002-worker-separado-com-polling.md), [ADR-005](./ADR-005-entrega-at-least-once.md)

## Contexto

Endpoints de clientes podem ficar temporariamente indisponíveis. Três
retentativas cobririam somente uma janela curta, enquanto retry indefinido
manteria eventos sem destino para sempre. Após o esgotamento, o time precisa de
evidência persistente para diagnóstico e reprocessamento.

Fontes: `[09:14]`–`[09:18]`, `[09:35]`–`[09:36]`.

## Decisão

Após a tentativa inicial falhar, realizar **cinco retentativas** com os atrasos
`1 minuto`, `5 minutos`, `30 minutos`, `2 horas` e `12 horas`. O contador
`retryCount` diferencia o primeiro envio das retentativas, eliminando a
ambiguidade entre “tentativa” e “retry”. Portanto, um evento pode produzir no
máximo seis chamadas HTTP: uma inicial e cinco reenvios.

Se a quinta retentativa falhar:

- copiar o evento, payload, motivo e horário para `webhook_dead_letter`;
- encerrar o processamento da linha original da outbox;
- permitir replay manual por
  `POST /api/v1/admin/webhooks/dead-letter/:id/replay`;
- restringir o replay à role `ADMIN` e registrar o usuário responsável.

A movimentação para DLQ e o encerramento da outbox devem ocorrer na mesma
transação. O replay cria um novo ciclo `PENDING` com `retryCount = 0`, mas
preserva no registro da DLQ a ligação com o evento original.

## Alternativas Consideradas

### Somente três tentativas

Descartada porque falharia cedo demais para clientes com manutenções de algumas
horas, já observadas pelo time.

### Retry indefinido

Descartada porque um cliente abandonado manteria eventos pendentes para sempre e
degradaria a outbox.

### Status `FAILED` apenas na outbox

Descartada em favor de tabela de DLQ separada, que mantém a leitura operacional
da outbox limpa e concentra as evidências para debug e replay.

## Consequências

### Positivas

- Cobre indisponibilidades transitórias por aproximadamente 14h36.
- Impede eventos eternamente pendentes.
- Mantém evidência de falhas permanentes e caminho de recuperação manual.

### Negativas

- Uma entrega pode permanecer atrasada por muitas horas.
- O mesmo evento pode chegar mais de uma vez.
- O replay exige operação administrativa e auditoria.
- Alertas por e-mail não existem nesta fase; o acompanhamento depende de
  histórico, logs e métricas.
