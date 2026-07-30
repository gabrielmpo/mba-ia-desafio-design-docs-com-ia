# ADR-003 — Retry com backoff e DLQ

- **Status:** Aceita
- **Decisores:** Larissa, Bruno e Diego

## Contexto

Endpoints dos clientes podem ficar indisponíveis ou responder lentamente. Uma falha transitória não deve provocar perda do evento, mas retries indefinidos manteriam eventos pendentes para sempre e prejudicariam a operação da outbox.

## Decisão

Após uma tentativa malsucedida, realizar até cinco novas tentativas nos intervalos:

1. 1 minuto;
2. 5 minutos;
3. 30 minutos;
4. 2 horas;
5. 12 horas.

Respostas não bem-sucedidas, erros de rede e timeout de dez segundos serão tratados como falha de entrega. Esgotadas as tentativas, o evento será copiado para `webhook_dead_letter`, com payload, motivo e timestamps, e deixará o fluxo normal da outbox.

O reprocessamento será manual por `POST /admin/webhooks/dead-letter/:id/replay`, protegido pela role `ADMIN` e auditado.

## Alternativas consideradas

### Três tentativas

Descartada porque não cobre indisponibilidades conhecidas de algumas horas.

### Retry indefinido

Descartado porque eventos de endpoints abandonados nunca seriam encerrados.

### Marcar falha permanente na própria outbox

Descartado em favor de uma tabela separada, que mantém a consulta de pendentes simples e preserva evidências para suporte.

## Consequências

### Positivas

- recuperação automática de falhas transitórias;
- janela de aproximadamente quinze horas;
- isolamento e rastreabilidade de falhas permanentes;
- replay controlado e auditável.

### Negativas

- a entrega pode permanecer atrasada por muitas horas;
- o endpoint pode receber duplicatas;
- DLQ exige armazenamento, operação e autorização administrativa;
- a semântica de quais códigos HTTP são retentáveis deve ser detalhada no FDD.

## Rastreabilidade

- `TRANSCRICAO.md`: `[09:14]`–`[09:19]`, `[09:35]`–`[09:36]`, `[09:42]`.
- Código relacionado: `src/middlewares/auth.middleware.ts` e padrão de autorização do projeto.

