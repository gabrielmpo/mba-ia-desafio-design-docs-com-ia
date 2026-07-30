# FDD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Autor** | Gabriel Oliveira |
| **Status** | Proposto para implementação |
| **Data** | 30/07/2026 |
| **Revisores** | Larissa (Tech Lead), Bruno (Engenharia de Pedidos), Diego (Engenharia de Plataforma), Sofia (Segurança) |
| **RFC relacionado** | [RFC — Sistema de Webhooks de Notificação de Pedidos](RFC.md) |
| **ADRs relacionados** | [ADR-001](adrs/ADR-001-outbox-transacional-no-mysql.md) a [ADR-007](adrs/ADR-007-reuso-dos-padroes-do-oms.md) |

## 1. Contexto e motivação técnica

O OMS precisa notificar sistemas de clientes quando um pedido muda de status. A aplicação atual é uma API Node.js 20 com Express, TypeScript, Prisma e MySQL. O método `OrderService.changeStatus` já concentra uma transação que valida a máquina de estados, altera o pedido, atualiza o estoque quando necessário e registra `OrderStatusHistory`.

A entrega do webhook não pode ocorrer dentro dessa transação: latência, timeout ou indisponibilidade do cliente passariam a bloquear uma mudança válida de pedido. Também não é aceitável registrar a notificação depois do commit, pois uma falha entre as duas operações deixaria um status confirmado sem o evento correspondente.

Este FDD detalha a implementação definida no RFC e nos ADRs: outbox transacional no MySQL, worker separado com polling, entrega at-least-once, retry com backoff, DLQ persistida e HMAC-SHA256 por endpoint.

Fontes principais: `DEC-01` a `DEC-26`, `RF-01` a `RF-17` e `RNF-01` a `RNF-15` do [inventário de evidências](INVENTARIO-EVIDENCIAS.md).

## 2. Objetivos técnicos

- Gravar o evento na mesma transação da mudança de status.
- Manter chamadas HTTP externas fora da transação de pedidos.
- Iniciar o processamento em até dois segundos após a criação, em condições normais.
- Entregar normalmente em menos de dez segundos.
- Preservar um snapshot imutável da transição que originou o evento.
- Entregar com semântica at-least-once e identificador estável para deduplicação.
- Recuperar falhas transitórias conforme os cinco intervalos de backoff decididos e encaminhar falhas permanentes à DLQ; confirmar a contagem total de chamadas.
- Permitir configuração, consulta de histórico, rotação de secret e replay administrativo.
- Reutilizar Prisma, Zod, `AppError`, autenticação, autorização, respostas paginadas e Pino.
- Manter API e worker implantáveis, reiniciáveis e observáveis separadamente.

## 3. Escopo e exclusões

### 3.1 Incluído

- CRUD de endpoints outbound associados a um customer.
- Filtro dos status de interesse de cada endpoint.
- Geração e rotação de secret exclusiva por endpoint.
- Evento `order.status_changed`.
- Snapshot do payload na outbox.
- Worker único, serial, em processo separado.
- Polling a cada dois segundos.
- Timeout HTTP de dez segundos.
- Política de tentativas com intervalos de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas; a inclusão da chamada inicial na contagem de cinco permanece aberta.
- Histórico de todas as tentativas.
- DLQ separada e replay manual por usuário `ADMIN`.
- HMAC-SHA256, HTTPS obrigatório e limite de payload de 64 KB.

### 3.2 Fora de escopo

- Webhooks inbound.
- Email de alerta ou fallback.
- Dashboard visual.
- Rate limiting de saída.
- Redis, broker dedicado ou outro componente de mensageria.
- Execução concorrente com múltiplos workers.
- Particionamento por `order_id`.
- Garantia exactly-once ou ordering global.
- Itens completos do pedido no payload.
- Política definitiva de arquivamento da outbox, do histórico e da DLQ.

Escala horizontal e retenção permanecem decisões futuras baseadas em métricas de produção.

## 4. Questões abertas que bloqueiam detalhes de implementação

A reunião não fechou os itens abaixo. Eles não são decisões deste FDD e não devem ser implementados como se fossem requisitos aprovados. Cada item precisa de validação do responsável indicado antes do início da tarefa correspondente.

| Questão aberta | Evidência disponível | Tratamento até a decisão |
| --- | --- | --- |
| Tamanho do lote | Polling de 2 segundos e single worker foram decididos; o lote não foi quantificado | Tornar o valor configurável, sem fixar default neste documento |
| Claim, locking e recuperação após queda | Lock pessimista foi citado apenas como alternativa futura para múltiplos workers em `[09:13]` | Não exigir `SKIP LOCKED`, lease ou estado `PROCESSING` na primeira fase; detalhar a recuperação antes de implementar o worker |
| Contagem de tentativas | `[09:17]` diz “5 tentativas” e também lista cinco intervalos (1m/5m/30m/2h/12h) | Confirmar se são cinco chamadas totais ou uma inicial mais cinco retries |
| Códigos HTTP retentáveis | Retry, backoff, timeout e DLQ foram decididos; a classificação por status não foi | Definir matriz de `3xx`, `4xx` e `5xx` antes de implementar o processor |
| Uso de jitter | Não discutido | Não tratar presença ou ausência de jitter como requisito |
| Proteção da secret em repouso | Secret por endpoint e rotação foram decididas; algoritmo de armazenamento não foi | Submeter a modelagem à revisão de Sofia antes da migration |
| Mecânica do grace period | A secret antiga deve permanecer válida por 24 horas, conforme `[09:21]`–`[09:22]` | Definir como o consumidor receberá/verificará a assinatura antiga; não assumir duas assinaturas no mesmo header |
| Formato de `X-Signature` e serialização | HMAC-SHA256 sobre o corpo foi decidido; encoding, prefixo e canonicalização não | Congelar o corpo por tentativa e publicar um vetor de teste após a definição do formato |
| Versionamento do evento | Não discutido | Não incluir `schema_version` como campo obrigatório nesta fase |
| Paginação do histórico | Foram pedidos os últimos 100 registros em `[09:34]`; parâmetros e defaults não foram definidos | Reutilizar o helper existente quando aplicável, deixando o contrato exato para validação |
| Roles do CRUD | Qualquer role autenticada em `[09:36]`–`[09:37]`; replay somente `ADMIN` | Aplicar apenas autenticação ao CRUD e `requireRole('ADMIN')` ao replay |
| Rate limiting | Adiado em `[09:38]`–`[09:39]` | Não implementar nesta fase; medir volume e falhas |

## 5. Arquitetura e componentes

O novo domínio ficará em `src/modules/webhooks` e seguirá a estrutura modular existente:

```text
src/
├── modules/
│   └── webhooks/
│       ├── webhook.controller.ts
│       ├── webhook.service.ts
│       ├── webhook.repository.ts
│       ├── webhook.routes.ts
│       ├── webhook.schemas.ts
│       ├── webhook.errors.ts
│       ├── webhook.event.ts
│       └── webhook.processor.ts
├── worker.ts
└── ...
```

Responsabilidades:

| Componente | Responsabilidade |
| --- | --- |
| `webhook.controller.ts` | Adaptar requests e responses HTTP e encaminhar erros com `next(err)` |
| `webhook.service.ts` | CRUD, rotação de secret, histórico e replay |
| `webhook.repository.ts` | Persistência de endpoints, outbox, deliveries e DLQ |
| `webhook.schemas.ts` | Schemas Zod para params, query e body |
| `webhook.errors.ts` | Erros de domínio com prefixo `WEBHOOK_*` |
| `webhook.event.ts` | Montagem determinística do snapshot e inserção transacional |
| `webhook.processor.ts` | Claim, entrega, HMAC, retry, DLQ e recuperação de lease |
| `src/worker.ts` | Bootstrap, loop de polling e graceful shutdown |

## 6. Modelo de dados

O modelo abaixo é uma proposta de implementação derivada das entidades decididas na reunião. Nomes de campos auxiliares, tamanhos de colunas e índices que não possuem timestamp ou equivalente no código são sugestões não normativas e precisam ser validados na migration review.

Os nomes abaixo seguem o mapeamento camelCase do Prisma para snake_case no MySQL.

### 6.1 `WebhookEndpoint`

Representa uma configuração outbound.

| Campo | Tipo | Regra |
| --- | --- | --- |
| `id` | `String @db.Char(36)` | UUID, chave primária |
| `customerId` | `String @db.Char(36)` | FK para `Customer` |
| `url` | `String @db.VarChar(2048)` | URL HTTPS válida |
| `active` | `Boolean` | `true` na criação |
| `secretProtected` | `String @db.Text` | Representação recuperável da secret atual; mecanismo a definir na revisão de segurança |
| `previousSecretProtected` | `String? @db.Text` | Representação recuperável da secret anterior durante a rotação |
| `previousSecretValidUntil` | `DateTime?` | Expiração após 24 horas |
| `createdAt` / `updatedAt` | `DateTime` | Auditoria temporal |
Índice inicial:

- `@@index([customerId, active])`.

A semântica de remoção física ou lógica não foi definida na reunião e deve ser confirmada antes da migration.

### 6.2 `WebhookEndpointStatus`

Relaciona endpoint e status de pedido. A chave composta `(webhookId, status)` evita duplicidade e permite filtrar assinantes antes de criar a outbox.

| Campo | Tipo | Regra |
| --- | --- | --- |
| `webhookId` | `String @db.Char(36)` | FK para `WebhookEndpoint` |
| `status` | `OrderStatus` | Status de interesse |

### 6.3 `WebhookOutbox`

Armazena um evento por endpoint interessado.

| Campo | Tipo | Regra |
| --- | --- | --- |
| `id` | `String @db.Char(36)` | UUID imutável e valor de `event_id` |
| `webhookId` | `String @db.Char(36)` | Endpoint de destino |
| `customerId` | `String @db.Char(36)` | Customer do pedido |
| `orderId` | `String @db.Char(36)` | Pedido que originou o evento |
| `eventType` | `String @db.VarChar(100)` | `order.status_changed` |
| `payloadJson` | `String @db.Text` | JSON serializado e imutável |
| `status` | `WebhookOutboxStatus` | No mínimo `PENDING`, `RETRY_SCHEDULED` e `DELIVERED`; estados de claim dependem da estratégia ainda aberta |
| `attemptCount` | `Int` | Número de chamadas HTTP já realizadas; limite final depende da confirmação da contagem |
| `nextAttemptAt` | `DateTime` | Momento a partir do qual o evento está elegível |
| `lastErrorCode` | `String? @db.VarChar(100)` | Último código operacional |
| `lastErrorMessage` | `String? @db.VarChar(1000)` | Mensagem sanitizada |
| `createdAt` / `updatedAt` | `DateTime` | Auditoria temporal |
| `deliveredAt` | `DateTime?` | Entrega confirmada |

Índices obrigatórios:

- `@@index([status, nextAttemptAt, createdAt])`;
- `@@index([orderId, createdAt])`;
- `@@index([webhookId, createdAt])`.

### 6.4 `WebhookDelivery`

Registra cada chamada HTTP, inclusive falhas.

| Campo | Tipo | Regra |
| --- | --- | --- |
| `id` | `String @db.Char(36)` | UUID |
| `eventId` | `String @db.Char(36)` | ID estável do evento |
| `webhookId` | `String @db.Char(36)` | Endpoint usado |
| `payloadJson` | `String @db.Text` | Snapshot enviado nesta tentativa |
| `attemptNumber` | `Int` | Sequência da chamada; limite depende da confirmação da contagem de tentativas |
| `outcome` | `WebhookDeliveryOutcome` | `SUCCESS` ou `FAILURE` |
| `httpStatus` | `Int?` | Ausente para erro de rede ou timeout |
| `errorCode` | `String? @db.VarChar(100)` | Código `WEBHOOK_*` |
| `responseBody` | `String? @db.Text` | Resposta registrada para histórico; limite e sanitização devem ser definidos antes da implementação |
| `durationMs` | `Int` | Duração da tentativa |
| `createdAt` | `DateTime` | Horário da tentativa |

Índices:

- `@@index([webhookId, createdAt])`;
- `@@index([eventId, attemptNumber])`.

### 6.5 `WebhookDeadLetter`

Preserva eventos removidos do fluxo normal.

| Campo | Tipo | Regra |
| --- | --- | --- |
| `id` | `String @db.Char(36)` | UUID do registro de DLQ |
| `eventId` | `String @db.Char(36)` | ID estável do evento original |
| `webhookId`, `customerId`, `orderId` | `String @db.Char(36)` | Contexto original |
| `eventType` | `String @db.VarChar(100)` | Tipo do evento |
| `payloadJson` | `String @db.Text` | Snapshot original |
| `attemptCount` | `Int` | Quantidade de chamadas realizadas |
| `reasonCode` | `String @db.VarChar(100)` | Código `WEBHOOK_*` |
| `reasonMessage` | `String @db.VarChar(1000)` | Mensagem sanitizada |
| `failedAt` | `DateTime` | Entrada na DLQ |
| `replayedAt` | `DateTime?` | Último replay |
| `replayedById` | `String? @db.Char(36)` | FK para o usuário `ADMIN` |

O replay não apaga o registro de DLQ. Ele marca `replayedAt` e `replayedById` e recria uma entrada `PENDING` na outbox preservando `eventId` e `payloadJson`. Assim, o consumidor continua deduplicando pelo identificador original.

## 7. Fluxos detalhados

### 7.1 Criação do evento na outbox

1. `OrderService.changeStatus` abre a transação existente.
2. O pedido é lido, a transição é validada por `canTransition` e os ajustes de estoque são executados.
3. O pedido é atualizado e `OrderStatusHistory` é criado.
4. Antes do commit, `publishOrderStatusChanged(tx, data)` é chamado com o mesmo `Prisma.TransactionClient`.
5. A função consulta endpoints ativos do `customerId` que assinam o `toStatus`.
6. Se não houver endpoint interessado, nenhuma linha de outbox é criada.
7. Para cada endpoint:
   - gera um UUID;
   - cria o payload em ordem de campos fixa;
   - serializa uma única vez com `JSON.stringify`;
   - valida `Buffer.byteLength(payloadJson, 'utf8') <= 65_536`;
   - insere a linha `PENDING`, com `nextAttemptAt = createdAt`.
8. Qualquer falha reverte status, histórico, estoque e outbox juntos.
9. O commit retorna normalmente para o endpoint de mudança de status já existente.

Assinantes são filtrados na inserção, não no envio. O payload nunca é reconstruído pelo worker.

### 7.2 Leitura pelo worker

Em cada ciclo da primeira fase:

1. Consultar eventos `PENDING` ou `RETRY_SCHEDULED` com `nextAttemptAt <= now`.
2. Ordenar por `createdAt ASC`, conforme `[09:12]`.
3. Limitar a consulta por um tamanho de lote configurável, cujo valor ainda precisa ser definido.
4. Processar serialmente no único worker decidido para a fase inicial.
5. Não manter transação aberta durante chamadas HTTP.

A reunião não definiu claim, lease nem recuperação de evento interrompido. `SELECT ... FOR UPDATE SKIP LOCKED`, status `PROCESSING` e timeout de lease são alternativas possíveis, não requisitos aprovados. A estratégia escolhida deve preservar at-least-once após reinício e ser registrada antes da implementação. O ordering é implícito no single worker ordenado por `createdAt`; não há garantia global nem desenho aprovado para múltiplos workers.

### 7.3 Entrega HTTP

Para cada evento:

1. Recarregar o endpoint e confirmar que continua ativo.
2. Recuperar a secret atual pelo mecanismo aprovado na revisão de segurança; durante o grace period, aplicar a mecânica de compatibilidade que ainda será definida.
3. Usar exatamente `payloadJson` como corpo, sem parse ou nova serialização.
4. Calcular HMAC-SHA256 sobre o corpo conforme o formato de assinatura que ainda será definido.
5. Efetuar `POST` com timeout de 10 segundos. O cliente HTTP e a política de redirects permanecem decisões de implementação abertas.
6. Registrar `WebhookDelivery` com duração, status, resposta e resultado.
7. Em resposta considerada bem-sucedida, marcar a outbox como `DELIVERED`.
8. Em falha classificada como retentável, reagendar ou mover à DLQ quando a política aprovada se esgotar.
9. Em falha classificada como permanente, mover à DLQ.
10. Tratar endpoint removido ou inativo conforme a semântica de remoção ainda a confirmar.

O worker não mantém transação de banco aberta durante o request HTTP.

### 7.4 Retry

`attemptCount` representa chamadas HTTP realizadas, inclusive as interrompidas por timeout. A reunião decidiu “5 tentativas” e os intervalos `1m/5m/30m/2h/12h`, mas não esclareceu se a tentativa inicial está incluída nessa contagem. Essa ambiguidade deve ser resolvida antes de codificar o limite.

| Ordem dos intervalos decididos | Aguardar antes da próxima execução |
| ---: | ---: |
| 1 | 1 minuto |
| 2 | 5 minutos |
| 3 | 30 minutos |
| 4 | 2 horas |
| 5 | 12 horas |

O timeout de 10 segundos, o backoff e o envio final à DLQ estão confirmados. A classificação de erros de rede e códigos HTTP como retentáveis ou permanentes, o comportamento de redirects e o uso de jitter não foram decididos. Até essa matriz ser aprovada, o FDD não atribui semântica definitiva a `3xx`, `4xx` ou `5xx`. O mesmo `event_id` e o snapshot do corpo são preservados nas novas tentativas.

### 7.5 DLQ e replay

Ao mover para a DLQ, uma única transação:

1. cria `WebhookDeadLetter` com o snapshot e o motivo;
2. remove a linha correspondente da outbox;
3. preserva os registros de `WebhookDelivery`.

No replay:

1. autenticar JWT;
2. exigir `requireRole('ADMIN')`;
3. buscar o item da DLQ;
4. verificar que não existe outbox ativa para o mesmo `eventId`;
5. criar nova outbox `PENDING`, `attemptCount = 0` e `nextAttemptAt = now`;
6. preencher `replayedAt` e `replayedById` no item da DLQ;
7. registrar log de auditoria com `requestId`, `eventId`, `deadLetterId` e `userId`;
8. responder `202 Accepted`.

O replay pode novamente terminar na DLQ. Nesse caso, um novo registro de DLQ é criado para preservar cada ciclo operacional.

## 8. Contrato do evento outbound

### 8.1 Corpo

```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "event_type": "order.status_changed",
  "timestamp": "2026-07-30T12:34:56.789Z",
  "order_id": "b1c7f329-837d-49f5-8d67-f8f2b9466e70",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "055b46d4-8d63-4dd9-b6bf-6d4ce3cd65f1",
  "total_cents": 15990
}
```

Regras:

- codificação UTF-8;
- `Content-Type: application/json`;
- os campos exibidos correspondem ao payload decidido em `[09:43]`; a estratégia exata de serialização para HMAC permanece aberta;
- `timestamp` é o instante de criação do evento em UTC/ISO 8601;
- limite máximo de 65.536 bytes;
- itens do pedido não são enviados;
- retries reutilizam os mesmos bytes.

### 8.2 Headers

```http
Content-Type: application/json
X-Event-Id: 550e8400-e29b-41d4-a716-446655440000
X-Webhook-Id: a4525fc4-85f7-46cb-8c53-a839fd8755e8
X-Timestamp: 2026-07-30T12:34:58.132Z
X-Signature: <hmac-sha256-em-formato-a-definir>
```

A secret anterior permanece válida por 24 horas após a rotação. A reunião não definiu se a compatibilidade será representada por duas assinaturas, headers separados ou outro mecanismo; portanto, o contrato exato de `X-Signature` deve ser aprovado por Segurança e acompanhado de vetor de teste. `X-Timestamp` registra o instante da tentativa; a deduplicação usa `X-Event-Id`.

## 9. Contratos públicos da API

Os métodos, recursos e regras de autenticação abaixo vêm da reunião. Paths completos, exemplos de status HTTP e detalhes de paginação que não aparecem literalmente na transcrição são propostas alinhadas aos padrões do código existente e devem ser confirmadas na API review.

Todas as rotas ficam sob `/api/v1`, recebem `Authorization: Bearer <jwt>` e usam o formato de erro do middleware central:

```json
{
  "error": {
    "code": "WEBHOOK_NOT_FOUND",
    "message": "Webhook not found"
  }
}
```

### 9.1 Criar endpoint — `POST /api/v1/webhooks`

Autorização: qualquer usuário autenticado, conforme `[09:36]`–`[09:37]`.

Request:

```json
{
  "customerId": "055b46d4-8d63-4dd9-b6bf-6d4ce3cd65f1",
  "url": "https://integracao.cliente.com/webhooks/orders",
  "statuses": ["SHIPPED", "DELIVERED"]
}
```

Response `201 Created`:

```json
{
  "id": "a4525fc4-85f7-46cb-8c53-a839fd8755e8",
  "customerId": "055b46d4-8d63-4dd9-b6bf-6d4ce3cd65f1",
  "url": "https://integracao.cliente.com/webhooks/orders",
  "statuses": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_Aq7...Yp2",
  "createdAt": "2026-07-30T12:00:00.000Z"
}
```

A secret é gerada pelo servidor com mecanismo criptograficamente seguro e exibida na resposta de criação. Parâmetros exatos de geração e política de reexibição devem ser aprovados na revisão de segurança.

Status possíveis: `201`, `400`, `401`, `404`, `409`.

### 9.2 Listar por customer — `GET /api/v1/webhooks`

Autorização: qualquer usuário autenticado, conforme `[09:36]`–`[09:37]`.

Request:

```http
GET /api/v1/webhooks?customerId=055b46d4-8d63-4dd9-b6bf-6d4ce3cd65f1
```

Response `200 OK`:

```json
{
  "data": [
    {
      "id": "a4525fc4-85f7-46cb-8c53-a839fd8755e8",
      "customerId": "055b46d4-8d63-4dd9-b6bf-6d4ce3cd65f1",
      "url": "https://integracao.cliente.com/webhooks/orders",
      "statuses": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-07-30T12:00:00.000Z",
      "updatedAt": "2026-07-30T12:00:00.000Z"
    }
  ],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "total": 1,
    "totalPages": 1
  }
}
```

Secrets nunca aparecem em listagens ou consultas.

Status possíveis: `200`, `400`, `401`, `404`.

### 9.3 Editar endpoint — `PATCH /api/v1/webhooks/:id`

Autorização: qualquer usuário autenticado, conforme `[09:36]`–`[09:37]`.

Request:

```json
{
  "url": "https://nova-integracao.cliente.com/orders",
  "statuses": ["PAID", "PROCESSING", "SHIPPED", "DELIVERED"],
  "active": true
}
```

Todos os campos são opcionais, mas o body deve conter ao menos um. `statuses` não pode ser vazio.

Response `200 OK`:

```json
{
  "id": "a4525fc4-85f7-46cb-8c53-a839fd8755e8",
  "customerId": "055b46d4-8d63-4dd9-b6bf-6d4ce3cd65f1",
  "url": "https://nova-integracao.cliente.com/orders",
  "statuses": ["PAID", "PROCESSING", "SHIPPED", "DELIVERED"],
  "active": true,
  "updatedAt": "2026-07-30T13:00:00.000Z"
}
```

Status possíveis: `200`, `400`, `401`, `404`, `409`.

### 9.4 Remover endpoint — `DELETE /api/v1/webhooks/:id`

Autorização: qualquer usuário autenticado, conforme `[09:36]`–`[09:37]`.

A reunião exige remoção, mas não definiu exclusão física ou lógica nem o destino dos eventos pendentes. O contrato final deve escolher essa semântica antes da migration.

Response: `204 No Content`.

Status possíveis: `204`, `400`, `401`, `404`.

### 9.5 Rotacionar secret — `POST /api/v1/webhooks/:id/rotate-secret`

Autorização: qualquer usuário autenticado, conforme `[09:36]`–`[09:37]`.

Request sem body.

Response `200 OK`:

```json
{
  "id": "a4525fc4-85f7-46cb-8c53-a839fd8755e8",
  "secret": "whsec_Qk9...Tf4",
  "previousSecretValidUntil": "2026-07-31T14:00:00.000Z"
}
```

A nova secret também é exibida somente uma vez. O comportamento de uma segunda rotação dentro do grace period ainda precisa ser definido.

Status possíveis: `200`, `401`, `404`, `409`.

### 9.6 Histórico — `GET /api/v1/webhooks/:id/deliveries`

Autorização: qualquer usuário autenticado, conforme `[09:36]`–`[09:37]`.

Request:

```http
GET /api/v1/webhooks/a4525fc4-85f7-46cb-8c53-a839fd8755e8/deliveries
```

Response `200 OK`:

```json
{
  "data": [
    {
      "id": "3f4117b3-5a9e-4ed8-b5b1-43c60c9c8c23",
      "eventId": "550e8400-e29b-41d4-a716-446655440000",
      "attemptNumber": 1,
      "outcome": "SUCCESS",
      "httpStatus": 204,
      "payload": {
        "event_id": "550e8400-e29b-41d4-a716-446655440000",
        "event_type": "order.status_changed",
        "timestamp": "2026-07-30T12:34:56.789Z",
        "order_id": "b1c7f329-837d-49f5-8d67-f8f2b9466e70",
        "order_number": "ORD-000123",
        "from_status": "PROCESSING",
        "to_status": "SHIPPED",
        "customer_id": "055b46d4-8d63-4dd9-b6bf-6d4ce3cd65f1",
        "total_cents": 15990
      },
      "responseBody": "",
      "responseTruncated": false,
      "durationMs": 184,
      "createdAt": "2026-07-30T12:34:58.316Z"
    }
  ],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "total": 1,
    "totalPages": 1
  }
}
```

Status possíveis: `200`, `400`, `401`, `404`.

### 9.7 Replay da DLQ — `POST /api/v1/admin/webhooks/dead-letter/:id/replay`

Role: somente `ADMIN`.

Request sem body.

Response `202 Accepted`:

```json
{
  "deadLetterId": "96456c4a-09a6-49eb-b0d1-b36a64c76446",
  "eventId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "PENDING",
  "replayedBy": "c9df368c-e709-4bbf-848a-fb509d303339",
  "replayedAt": "2026-07-30T15:00:00.000Z"
}
```

Status possíveis: `202`, `400`, `401`, `403`, `404`, `409`.

## 10. Matriz de erros

Os códigos seguem o prefixo `WEBHOOK_` decidido em `[09:28]`–`[09:30]`. Códigos específicos e status HTTP são propostas para revisão, exceto quando vinculados diretamente a uma validação ou decisão citada.

Erros de forma genérica continuam usando `VALIDATION_ERROR`, `UNAUTHORIZED` e `FORBIDDEN` dos middlewares compartilhados. Erros de domínio e operação do módulo usam `WEBHOOK_*`.

| Código | HTTP | Condição | Tratamento |
| --- | ---: | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | Endpoint não existe ou foi removido | Retornar erro |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `customerId` inexistente | Retornar erro |
| `WEBHOOK_INVALID_URL` | 400 | URL inválida ou sem HTTPS | Retornar detalhes de validação |
| `WEBHOOK_INVALID_STATUSES` | 400 | Lista vazia, duplicada ou com status inválido | Retornar detalhes |
| `WEBHOOK_DUPLICATE_ENDPOINT` | 409 | Mesmo customer e URL ativa já cadastrados | Não criar duplicata |
| `WEBHOOK_SECRET_ENCRYPTION_FAILED` | 500 | Falha ao proteger ou recuperar secret | Não expor material criptográfico |
| `WEBHOOK_SECRET_ROTATION_CONFLICT` | 409 | Rotação concorrente | Cliente pode repetir a operação |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Snapshot excede 64 KB | Reverter a mudança de status |
| `WEBHOOK_DELIVERY_TIMEOUT` | — | Request excede 10 segundos | Registrar e aplicar retry |
| `WEBHOOK_DELIVERY_NETWORK_ERROR` | — | DNS, TLS ou conexão falhou | Registrar e aplicar retry |
| `WEBHOOK_DELIVERY_HTTP_ERROR` | — | Resposta HTTP não classificada como sucesso | Aplicar a matriz retentável/permanente depois de aprovada |
| `WEBHOOK_RETRY_EXHAUSTED` | — | Quantidade aprovada de tentativas foi esgotada | Mover à DLQ |
| `WEBHOOK_ENDPOINT_INACTIVE` | — | Endpoint inativo antes da chamada | Aplicar a semântica de remoção ainda a confirmar |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Item da DLQ inexistente | Retornar erro |
| `WEBHOOK_DEAD_LETTER_ALREADY_QUEUED` | 409 | Já há outbox ativa para o mesmo evento | Não duplicar replay |

Mensagens persistidas e retornadas não devem incluir secrets, headers de autenticação nem stack traces.

## 11. Segurança e gestão de secrets

- Gerar secrets no servidor por mecanismo criptograficamente seguro; tamanho, codificação e prefixo dependem da revisão de segurança.
- Proteger a secret em repouso com mecanismo recuperável aprovado por Segurança; algoritmo ainda aberto.
- Carregar o material necessário ao mecanismo de proteção por configuração externa ao MySQL, após definição na revisão de segurança.
- Retornar a secret somente na criação e na rotação.
- Nunca incluir secrets no histórico, logs ou erros.
- Acrescentar ao redaction do Pino: `*.secret`, `*.secretCiphertext`, `*.previousSecretCiphertext` e `req.body.secret`.
- Exigir protocolo `https:` no schema e no service.
- Definir explicitamente a política de redirects na matriz de falhas antes da implementação.
- Comparações de assinatura no exemplo de integração do cliente devem usar função constant-time.
- Agendar pelo menos dois dias úteis de revisão de Sofia antes do deploy.

A política completa de proteção contra SSRF e resolução para endereços privados deve ser validada na revisão de segurança. Até essa decisão, a implementação não deve ser liberada em produção apenas com a checagem de protocolo.

## 12. Resiliência

- **Atomicidade:** status, histórico, estoque e outbox compartilham a transação.
- **Desacoplamento:** nenhuma chamada externa ocorre na API de pedidos.
- **Timeout:** 10 segundos por tentativa com `AbortController`.
- **Retry:** aplicar os cinco intervalos decididos após confirmar se a chamada inicial integra a contagem de cinco tentativas.
- **DLQ:** falhas permanentes ou esgotadas saem da consulta normal.
- **At-least-once:** falha após o cliente processar e antes do commit local pode gerar duplicata.
- **Recuperação:** definir como eventos interrompidos voltam ao fluxo sem violar at-least-once.
- **Shutdown:** ao receber `SIGINT` ou `SIGTERM`, o worker para novos claims, aguarda a tentativa atual até o timeout e desconecta o Prisma.
- **Falha de banco no polling:** registrar erro, aguardar o próximo intervalo e tentar novamente sem encerrar o processo.
- **Ordering:** no single worker, processar serialmente por `createdAt`; desenho para múltiplos workers fica fora de escopo.
- **Endpoint removido:** não criar novos eventos e cancelar os ainda não enviados.

## 13. Observabilidade

### 13.1 Métricas

| Métrica | Tipo | Labels | Uso |
| --- | --- | --- | --- |
| `webhook_outbox_pending` | gauge | — | Tamanho do backlog elegível |
| `webhook_outbox_oldest_age_seconds` | gauge | — | Lag do evento mais antigo |
| `webhook_delivery_total` | counter | `outcome`, `status_class` | Volume e taxa de sucesso |
| `webhook_delivery_duration_ms` | histogram | `outcome` | Latência do endpoint |
| `webhook_retry_scheduled_total` | counter | `attempt_number`, `reason` | Pressão de retries |
| `webhook_dead_letter_total` | counter | `reason` | Falhas permanentes |
| `webhook_replay_total` | counter | `outcome` | Operação administrativa |
| `webhook_payload_size_bytes` | histogram | — | Aproximação do limite de 64 KB |

O código-base ainda não possui biblioteca ou endpoint de métricas. Na primeira versão, esses valores devem ser emitidos como eventos Pino estruturados aptos a métricas derivadas. A adoção de um exporter dedicado pode ser feita sem alterar a semântica definida aqui.

### 13.2 Logs estruturados

Eventos mínimos:

- `webhook_event_enqueued`;
- `webhook_batch_claimed`;
- `webhook_delivery_started`;
- `webhook_delivery_succeeded`;
- `webhook_retry_scheduled`;
- `webhook_moved_to_dead_letter`;
- `webhook_dead_letter_replayed`;
- `webhook_processing_recovered` (após definição da estratégia de recuperação);
- `webhook_worker_started`;
- `webhook_worker_stopped`.

Campos comuns: `eventId`, `webhookId`, `customerId`, `orderId`, `attemptNumber`, `durationMs`, `httpStatus`, `errorCode` e `requestId` quando houver. Não registrar payload completo por padrão; o payload já está persistido para consulta autorizada.

### 13.3 Tracing e correlação

O projeto não possui tracing distribuído. Na primeira versão:

- `event_id` é o identificador de correlação ponta a ponta;
- a API registra `requestId` e `eventId` na criação da outbox;
- worker, delivery e DLQ mantêm o mesmo `eventId`;
- `X-Event-Id` propaga a correlação ao consumidor;
- uma futura instrumentação OpenTelemetry deverá usar `eventId`, `webhookId` e `orderId` como atributos dos spans.

Essa correlação permite reconstruir o fluxo sem introduzir um backend de tracing nesta fase.

### 13.4 Alertas operacionais

Recomenda-se alertar quando:

- o evento mais antigo pendente ultrapassar 10 segundos;
- a taxa de falha superar 5% em janela de 5 minutos;
- houver crescimento contínuo da DLQ;
- o worker não emitir heartbeat por mais de 30 segundos;
- o número de eventos recuperados após interrupção for maior que zero de forma recorrente.

Os limiares devem ser recalibrados após o piloto, pois o volume real permanece questão aberta.

## 14. Integração com o sistema existente

| Caminho real | Alteração e integração |
| --- | --- |
| `src/modules/orders/order.service.ts` | Chamar `publishOrderStatusChanged(tx, ...)` dentro de `changeStatus`, depois do update e do histórico e antes do commit. Reutilizar o mesmo `Prisma.TransactionClient`; falha na outbox causa rollback total. |
| `src/modules/orders/order.status.ts` | Reutilizar `OrderStatus` e `canTransition`; os valores aceitos em `statuses`, `from_status` e `to_status` vêm do mesmo enum. |
| `src/modules/orders/order.routes.ts` | Não altera o contrato de `PATCH /orders/:id/status`; a geração do evento é efeito transacional interno. |
| `src/modules/orders/order.controller.ts` | Mantém o padrão `try/catch` e `next(err)`; erros da outbox chegam ao middleware central sem tratamento paralelo. |
| `src/modules/orders/order.schemas.ts` | Serve de referência para `z.nativeEnum(OrderStatus)` e UUIDs nos novos schemas. |
| `prisma/schema.prisma` | Adicionar enums, modelos, FKs e índices de endpoint, status, outbox, deliveries e DLQ; manter UUID `@db.Char(36)` e nomes de tabela em snake_case. |
| `src/app.ts` | Construir repository, service e controller de webhooks com o mesmo `PrismaClient` da API. |
| `src/routes/index.ts` | Montar `buildWebhookRouter` em `/webhooks` e a rota administrativa em `/admin/webhooks`. |
| `src/middlewares/auth.middleware.ts` | Aplicar `authenticate` em todo o módulo e `requireRole('ADMIN')` no replay. |
| `src/middlewares/validate.middleware.ts` | Validar body, params e query com Zod antes do controller. |
| `src/middlewares/error.middleware.ts` | Reutilizar o tratamento de `AppError`, Zod e Prisma e o formato `{ error: { code, message, details } }`. |
| `src/shared/errors/index.ts` | Exportar os novos erros definidos em `webhook.errors.ts`, todos derivados de `AppError`. |
| `src/shared/http/response.ts` | Reutilizar `paginated` no GET de endpoints e deliveries. |
| `src/shared/logger/index.ts` | Reutilizar Pino e ampliar a lista de redaction para secrets. |
| `src/config/database.ts` | `src/worker.ts` chama `createPrismaClient()` para obter instância própria no processo separado. |
| `src/config/env.ts` | Validar as novas variáveis do worker e a chave de criptografia. |
| `src/server.ts` | Manter como entry point exclusivo da API; seu ciclo de vida não inicia o worker. |
| `package.json` | Adicionar scripts `worker:dev` e `worker`; Node 20 fornece `fetch`, `AbortController` e `crypto`, sem cliente HTTP adicional. |
| `tsconfig.build.json` | Já inclui `src/**/*.ts`, portanto compilará `src/worker.ts` para `dist/worker.js`. |

## 15. Configuração e execução

Variáveis:

| Variável | Obrigatória | Default | Regra |
| --- | --- | --- | --- |
| Configuração de proteção da secret | a definir | — | Nome, formato e origem dependem da revisão de segurança |
| `WEBHOOK_POLL_INTERVAL_MS` | não | `2000` | Manter 2000 em produção nesta fase |
| `WEBHOOK_BATCH_SIZE` | sim | sem default definido | Inteiro positivo; valor depende de validação |
| `WEBHOOK_HTTP_TIMEOUT_MS` | não | `10000` | Manter 10000 em produção nesta fase |

Scripts:

```json
{
  "scripts": {
    "worker:dev": "tsx watch --env-file=.env src/worker.ts",
    "worker": "node --env-file=.env dist/worker.js"
  }
}
```

API e worker usam a mesma `DATABASE_URL`, mas processos e pools Prisma separados.

## 16. Compatibilidade e evolução

- Node.js `>=20`, TypeScript `5.6`, Prisma `5.22` e MySQL `8.0` permanecem suportados.
- A mudança exige migration antes de habilitar a criação de eventos.
- O contrato de `PATCH /api/v1/orders/:id/status` não muda.
- A estratégia de versionamento do payload permanece aberta.
- Remoção, renomeação ou alteração semântica exige nova versão de schema e estratégia de convivência.
- O worker deve ser implantado somente após a migration e pode permanecer desabilitado enquanto a API começa a preencher a outbox.
- Rollback do worker não afeta a API; eventos permanecem persistidos.
- Rollback da API para versão que não gera eventos não remove tabelas imediatamente.

## 17. Estratégia de testes

### 17.1 Unitários

- preservação do snapshot e limite de 64 KB; estratégia de serialização ainda aberta;
- assinatura HMAC sobre os bytes exatos;
- compatibilidade da secret anterior durante o grace period, após definição do contrato;
- proteção e recuperação da secret pelo mecanismo aprovado por Segurança;
- validação HTTPS e status de pedido;
- classificação de erros retentáveis e permanentes;
- cálculo dos cinco intervalos;
- redaction de secrets;
- transições de estado da outbox.

### 17.2 Integração com MySQL/Prisma

- status, histórico, estoque e outbox commitam juntos;
- falha de inserção na outbox reverte toda a mudança;
- nenhum assinante produz zero eventos;
- dois endpoints interessados produzem dois eventos com IDs distintos;
- filtro de status ocorre antes da inserção;
- claim respeita ordem por `createdAt`;
- evento posterior do mesmo pedido aguarda o anterior;
- recuperação após interrupção segue a estratégia que ainda será aprovada;
- DLQ e remoção da outbox são atômicas;
- replay preserva `eventId` e audita o usuário.

### 17.3 Contrato HTTP

- CRUD disponível a qualquer role autenticada; replay restrito a `ADMIN`;
- replay negado para `OPERATOR`;
- secret visível somente na criação e rotação;
- paginação no formato compartilhado;
- todos os endpoints retornam status e shape documentados;
- erros do módulo apresentam códigos `WEBHOOK_*`;
- URL HTTP é recusada.

### 17.4 Worker e ponta a ponta

Usar servidor HTTP controlado nos testes para simular:

- `2xx`;
- timeout acima de 10 segundos;
- encerramento de conexão;
- classificação de códigos HTTP conforme matriz ainda a aprovar;
- queda do worker antes de persistir sucesso;
- resposta maior que o limite de histórico;
- duplicata com o mesmo `X-Event-Id`;
- rotação de secret durante retries;
- remoção do endpoint com eventos pendentes.

## 18. Critérios de aceite técnicos

- [ ] Migration cria modelos, relações e índices descritos.
- [ ] `changeStatus` e criação da outbox são atomicamente confirmados ou revertidos.
- [ ] Chamadas externas nunca ocorrem dentro da transação de pedidos.
- [ ] Apenas endpoints ativos e interessados no novo status geram eventos.
- [ ] Snapshot é imutável, não contém itens e não excede 64 KB.
- [ ] Worker roda em processo e `PrismaClient` separados da API.
- [ ] Polling ocorre a cada dois segundos com lote inicial de 20.
- [ ] Processamento no single worker é serial e ordenado por `createdAt`, dentro da limitação documentada.
- [ ] Cada chamada usa timeout de dez segundos.
- [ ] Falhas retentáveis seguem 1m/5m/30m/2h/12h.
- [ ] Após a quantidade de tentativas aprovada e os intervalos decididos, o evento vai para a DLQ.
- [ ] Retries e replay preservam `event_id` e payload.
- [ ] Request outbound inclui os quatro headers definidos.
- [ ] HMAC é calculado sobre os bytes enviados.
- [ ] Secret é exclusiva, criptografada, exibida uma vez e rotacionável com 24 horas de graça.
- [ ] Histórico contém sucesso/falha, payload, resposta e duração.
- [ ] Replay exige `ADMIN` e registra o usuário.
- [ ] Logs não expõem secret ou header de autenticação.
- [ ] Métricas, logs e correlação por `eventId` estão disponíveis.
- [ ] Testes unitários, integração, contrato e ponta a ponta passam.
- [ ] Revisão de segurança de dois dias úteis é concluída antes do deploy.

## 19. Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Crescimento da outbox e do histórico | Média | Alto | Índices, lote pequeno, métricas de backlog e política de retenção futura |
| Single worker não sustentar picos | Média | Alto | Monitorar lag; planejar locking/particionamento quando os dados justificarem |
| Endpoint lento bloquear o processamento serial | Alta | Médio | Timeout de 10 segundos e backoff decidido |
| Duplicata gerar efeito repetido no cliente | Média | Alto | `X-Event-Id`, documentação e testes de idempotência |
| Vazamento de secret | Baixa | Alto | Redaction, rotação, proteção em repouso a definir e revisão de segurança |
| SSRF por URL cadastrada | Média | Alto | HTTPS obrigatório e definição da política de redes permitidas antes da produção |
| Evento posterior ficar bloqueado durante retry do mesmo pedido | Média | Médio | Trade-off explícito para preservar ordering; medir lag por pedido |
| Falha após recebimento remoto e antes do commit local | Baixa | Médio | Semântica at-least-once e deduplicação no consumidor |
| Resposta externa conter dados sensíveis | Média | Médio | Acesso autenticado ao histórico, truncamento e futura política de retenção |

## 20. Rollout

1. Criar migration e modelos Prisma.
2. Implementar módulo, contratos e testes de CRUD sem habilitar geração de eventos.
3. Integrar `changeStatus` e validar rollback transacional.
4. Implantar API com geração controlada da outbox.
5. Implantar worker desabilitado e validar conexão, health e observabilidade.
6. Habilitar worker em ambiente de teste com endpoints simulados.
7. Executar revisão de segurança de HMAC, secrets, URLs e logs.
8. Fazer piloto com um cliente.
9. Habilitar progressivamente os demais clientes e acompanhar lag, falhas e DLQ.

## 21. Rastreabilidade resumida

| Item do FDD | Origem principal |
| --- | --- |
| Outbox e integração transacional | `DEC-01`, `DEC-02`, `RNF-02`, `RNF-03`, `COD-01` a `COD-08`; `[09:03]`–`[09:10]`, `[09:40]`–`[09:41]` |
| Worker, polling e single worker | `DEC-03` a `DEC-09`, `RNF-01`; `[09:08]`–`[09:14]` |
| Retry e DLQ | `DEC-10` a `DEC-14`, `RF-11`, `RF-12`, `RNF-04` a `RNF-06`; `[09:14]`–`[09:19]`, `[09:35]`–`[09:36]` |
| HMAC, HTTPS e secrets | `DEC-15` a `DEC-19`, `RF-13`, `RF-16`, `RNF-07` a `RNF-10`; `[09:19]`–`[09:24]` |
| At-least-once | `DEC-20`, `DEC-21`, `RNF-11`; `[09:24]`–`[09:26]` |
| CRUD e histórico | `RF-02` a `RF-10`; `[09:31]`–`[09:37]` |
| Payload e headers | `DEC-23`, `DEC-25`, `DEC-26`, `RF-14` a `RF-17`, `RNF-12`, `RNF-13`; `[09:42]`–`[09:45]`, `[09:51]`–`[09:52]` |
| Reuso do OMS | `DEC-24`, `RNF-14`, `COD-09` a `COD-20`; `[09:27]`–`[09:30]` |
| Exclusões | `EXC-01` a `EXC-07`; `[09:37]`–`[09:40]` |
| Prazo e revisão | `RNF-15`, `CTX-13`, `CTX-14`; `[09:45]`–`[09:47]` |
