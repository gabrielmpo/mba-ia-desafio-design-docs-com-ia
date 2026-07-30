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
- Recuperar falhas transitórias com cinco retries e encaminhar falhas permanentes à DLQ.
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
- Cinco retries com backoff de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas.
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

## 4. Decisões de implementação para questões abertas do RFC

As escolhas abaixo tornam a primeira versão implementável. Elas detalham a solução, mas não criam novos requisitos de produto.

| Questão | Decisão da primeira versão | Justificativa |
| --- | --- | --- |
| Tamanho do lote | 20 eventos, configurável por `WEBHOOK_BATCH_SIZE` entre 1 e 100 | Lote pequeno reduz tempo de lock e é compatível com o single worker |
| Claim e locking | Claim transacional com `SELECT ... FOR UPDATE SKIP LOCKED`, status `PROCESSING` e lease de 60 segundos | Impede duplo claim e recupera eventos após queda do processo |
| Códigos retentáveis | Rede, timeout, `408`, `425`, `429` e `5xx` | São falhas potencialmente transitórias |
| Falhas permanentes | Redirecionamentos e demais `4xx` vão diretamente para a DLQ | Repetição normalmente não corrige URL ou requisição inválida |
| Jitter | Não usar na primeira versão | Há somente um worker; preserva os intervalos aprovados na reunião |
| Secret em repouso | AES-256-GCM com chave externa ao banco | A secret precisa ser recuperável para assinar; hash irreversível não atende |
| Grace period | Por 24 horas, `X-Signature` contém assinaturas feitas com a secret atual e com a anterior | Permite migração sem interromper consumidores que ainda usam a secret antiga |
| Formato da assinatura | `sha256=<hex-lowercase>` sobre os bytes UTF-8 exatos do snapshot | Evita divergência por reserialização do JSON |
| Versionamento do evento | Campo `schema_version: 1`; mudanças aditivas mantêm a versão, breaking changes criam nova versão | Torna a evolução explícita |
| Paginação de deliveries | `page=1`, `pageSize=20`, máximo 100 | Reutiliza o padrão já existente no OMS |
| Roles do CRUD | Qualquer usuário autenticado; replay somente `ADMIN` | Decisão explícita da reunião em `[09:35]`–`[09:37]` |
| Rate limiting | Não implementar; observar volume, lag e taxa de falha | Item adiado em `[09:38]`–`[09:39]` |

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

Os nomes abaixo seguem o mapeamento camelCase do Prisma para snake_case no MySQL.

### 6.1 `WebhookEndpoint`

Representa uma configuração outbound.

| Campo | Tipo | Regra |
| --- | --- | --- |
| `id` | `String @db.Char(36)` | UUID, chave primária |
| `customerId` | `String @db.Char(36)` | FK para `Customer` |
| `url` | `String @db.VarChar(2048)` | URL HTTPS válida |
| `active` | `Boolean` | `true` na criação |
| `secretCiphertext` | `String @db.Text` | Secret atual criptografada |
| `secretIv` | `String @db.VarChar(64)` | IV da secret atual |
| `secretAuthTag` | `String @db.VarChar(64)` | Tag GCM da secret atual |
| `previousSecretCiphertext` | `String? @db.Text` | Secret anterior durante a rotação |
| `previousSecretIv` | `String? @db.VarChar(64)` | IV da secret anterior |
| `previousSecretAuthTag` | `String? @db.VarChar(64)` | Tag GCM da secret anterior |
| `previousSecretValidUntil` | `DateTime?` | Expiração após 24 horas |
| `createdAt` / `updatedAt` | `DateTime` | Auditoria temporal |
| `deletedAt` | `DateTime?` | Exclusão lógica |

Índices:

- `@@index([customerId, active])`;
- `@@index([deletedAt])`.

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
| `status` | `WebhookOutboxStatus` | `PENDING`, `PROCESSING`, `RETRY_SCHEDULED`, `DELIVERED` ou `CANCELLED` |
| `attemptCount` | `Int` | Número de chamadas HTTP já realizadas |
| `nextAttemptAt` | `DateTime` | Momento a partir do qual o evento está elegível |
| `lockedAt` | `DateTime?` | Início do lease |
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
| `attemptNumber` | `Int` | 1 para a tentativa inicial; 2 a 6 para retries |
| `outcome` | `WebhookDeliveryOutcome` | `SUCCESS` ou `FAILURE` |
| `httpStatus` | `Int?` | Ausente para erro de rede ou timeout |
| `errorCode` | `String? @db.VarChar(100)` | Código `WEBHOOK_*` |
| `responseBody` | `String? @db.Text` | Primeiros 64 KB da resposta |
| `responseTruncated` | `Boolean` | Indica truncamento |
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

### 7.2 Claim pelo worker

Em cada ciclo:

1. Recuperar leases abandonados: eventos `PROCESSING` com `lockedAt < now - 60s` voltam ao estado anterior elegível.
2. Abrir transação curta.
3. Selecionar até `WEBHOOK_BATCH_SIZE` eventos:
   - status `PENDING` ou `RETRY_SCHEDULED`;
   - `nextAttemptAt <= now`;
   - endpoint ativo;
   - sem evento anterior ainda não terminal para o mesmo `orderId`;
   - ordem por `createdAt ASC`;
   - lock `FOR UPDATE SKIP LOCKED`.
4. Alterar os selecionados para `PROCESSING` e preencher `lockedAt`.
5. Commitar o claim antes de qualquer chamada externa.
6. Processar serialmente os eventos reclamados.

Embora a primeira versão tenha somente um worker, o claim transacional elimina duplo processamento acidental e facilita recuperação após reinício. O processamento serial e o bloqueio de eventos posteriores preservam a ordem por pedido enquanto uma transição anterior estiver pendente ou em retry.

### 7.3 Entrega HTTP

Para cada evento:

1. Recarregar o endpoint e confirmar que continua ativo.
2. Descriptografar a secret atual; carregar também a anterior se ainda estiver no grace period.
3. Usar exatamente `payloadJson` como corpo, sem parse ou nova serialização.
4. Calcular HMAC-SHA256 para cada secret válida.
5. Efetuar `POST` com `fetch`, `redirect: 'manual'` e `AbortController` de 10 segundos.
6. Registrar `WebhookDelivery` com duração, status, resposta e resultado.
7. Em `2xx`, marcar a outbox como `DELIVERED`.
8. Em falha retentável, reagendar ou mover à DLQ caso tenham sido esgotados os retries.
9. Em falha permanente, mover imediatamente à DLQ.
10. Se o endpoint foi removido, marcar o evento `CANCELLED` sem chamada externa.

O worker não mantém transação de banco aberta durante o request HTTP.

### 7.4 Retry

`attemptCount` representa chamadas HTTP concluídas ou interrompidas por timeout. Há uma tentativa inicial e cinco retries, totalizando no máximo seis chamadas.

| Chamada que falhou | Próxima ação |
| ---: | --- |
| 1 — tentativa inicial | Retry 1 em 1 minuto |
| 2 — retry 1 | Retry 2 em 5 minutos |
| 3 — retry 2 | Retry 3 em 30 minutos |
| 4 — retry 3 | Retry 4 em 2 horas |
| 5 — retry 4 | Retry 5 em 12 horas |
| 6 — retry 5 | Mover para DLQ |

Falhas retentáveis:

- erro de DNS, conexão ou TLS;
- conexão encerrada sem resposta;
- timeout de 10 segundos;
- HTTP `408`, `425`, `429`;
- HTTP `500` a `599`.

Falhas permanentes:

- HTTP `300` a `399`, pois redirects não são seguidos;
- HTTP `400` a `499`, exceto `408`, `425` e `429`;
- endpoint removido ou configuração inválida.

O backoff não usa jitter na primeira versão. O mesmo `event_id` e o mesmo corpo são mantidos em todas as tentativas.

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
  "schema_version": 1,
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
- chaves sempre na ordem exibida;
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
X-Signature: sha256=7f1c...e029
```

Durante as 24 horas posteriores à rotação:

```http
X-Signature: sha256=<assinatura-atual>,sha256=<assinatura-anterior>
```

O consumidor divide o valor por vírgula, calcula o HMAC sobre os bytes recebidos e aceita se qualquer assinatura corresponder em comparação constant-time. `X-Timestamp` registra o instante da tentativa; a deduplicação deve usar `X-Event-Id`.

## 9. Contratos públicos da API

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

Roles: `ADMIN` e `OPERATOR`.

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

A secret é gerada com CSPRNG, tem pelo menos 256 bits de entropia e é exibida somente nesta resposta.

Status possíveis: `201`, `400`, `401`, `404`, `409`.

### 9.2 Listar por customer — `GET /api/v1/webhooks`

Roles: `ADMIN` e `OPERATOR`.

Request:

```http
GET /api/v1/webhooks?customerId=055b46d4-8d63-4dd9-b6bf-6d4ce3cd65f1&page=1&pageSize=20
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

Roles: `ADMIN` e `OPERATOR`.

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

Roles: `ADMIN` e `OPERATOR`.

Realiza exclusão lógica, define `active = false` e `deletedAt = now`. O worker cancela eventos ainda não enviados para esse endpoint.

Response: `204 No Content`.

Status possíveis: `204`, `400`, `401`, `404`.

### 9.5 Rotacionar secret — `POST /api/v1/webhooks/:id/rotate-secret`

Roles: `ADMIN` e `OPERATOR`.

Request sem body.

Response `200 OK`:

```json
{
  "id": "a4525fc4-85f7-46cb-8c53-a839fd8755e8",
  "secret": "whsec_Qk9...Tf4",
  "previousSecretValidUntil": "2026-07-31T14:00:00.000Z"
}
```

A nova secret também é exibida somente uma vez. Uma nova rotação substitui o par anterior e reinicia o grace period.

Status possíveis: `200`, `401`, `404`, `409`.

### 9.6 Histórico — `GET /api/v1/webhooks/:id/deliveries`

Roles: `ADMIN` e `OPERATOR`.

Request:

```http
GET /api/v1/webhooks/a4525fc4-85f7-46cb-8c53-a839fd8755e8/deliveries?page=1&pageSize=20
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
        "schema_version": 1,
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
| `WEBHOOK_DELIVERY_RETRYABLE_HTTP_ERROR` | — | `408`, `425`, `429` ou `5xx` | Registrar e aplicar retry |
| `WEBHOOK_DELIVERY_PERMANENT_HTTP_ERROR` | — | Redirect ou demais `4xx` | Mover à DLQ |
| `WEBHOOK_RETRY_EXHAUSTED` | — | Sexta chamada falhou | Mover à DLQ |
| `WEBHOOK_ENDPOINT_INACTIVE` | — | Endpoint removido antes da chamada | Cancelar evento |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Item da DLQ inexistente | Retornar erro |
| `WEBHOOK_DEAD_LETTER_ALREADY_QUEUED` | 409 | Já há outbox ativa para o mesmo evento | Não duplicar replay |

Mensagens persistidas e retornadas não devem incluir secrets, headers de autenticação nem stack traces.

## 11. Segurança e gestão de secrets

- Gerar secrets com `crypto.randomBytes(32)` e codificação base64url, prefixadas por `whsec_`.
- Criptografar em repouso com AES-256-GCM.
- Carregar a chave de 32 bytes a partir de `WEBHOOK_SECRET_ENCRYPTION_KEY`; nunca armazená-la no MySQL.
- Retornar a secret somente na criação e na rotação.
- Nunca incluir secrets no histórico, logs ou erros.
- Acrescentar ao redaction do Pino: `*.secret`, `*.secretCiphertext`, `*.previousSecretCiphertext` e `req.body.secret`.
- Exigir protocolo `https:` no schema e no service.
- Não seguir redirects.
- Comparações de assinatura no exemplo de integração do cliente devem usar função constant-time.
- Agendar pelo menos dois dias úteis de revisão de Sofia antes do deploy.

A política completa de proteção contra SSRF e resolução para endereços privados deve ser validada na revisão de segurança. Até essa decisão, a implementação não deve ser liberada em produção apenas com a checagem de protocolo.

## 12. Resiliência

- **Atomicidade:** status, histórico, estoque e outbox compartilham a transação.
- **Desacoplamento:** nenhuma chamada externa ocorre na API de pedidos.
- **Timeout:** 10 segundos por tentativa com `AbortController`.
- **Retry:** cinco retries determinísticos depois da tentativa inicial.
- **DLQ:** falhas permanentes ou esgotadas saem da consulta normal.
- **At-least-once:** falha após o cliente processar e antes do commit local pode gerar duplicata.
- **Lease:** itens `PROCESSING` há mais de 60 segundos voltam ao fluxo.
- **Shutdown:** ao receber `SIGINT` ou `SIGTERM`, o worker para novos claims, aguarda a tentativa atual até o timeout e desconecta o Prisma.
- **Falha de banco no polling:** registrar erro, aguardar o próximo intervalo e tentar novamente sem encerrar o processo.
- **Ordering:** serial por `createdAt`, bloqueando eventos posteriores do mesmo pedido enquanto houver anterior não terminal.
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
- `webhook_processing_lease_recovered`;
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
- o número de leases recuperados for maior que zero de forma recorrente.

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
| `WEBHOOK_SECRET_ENCRYPTION_KEY` | sim | — | Base64 de 32 bytes |
| `WEBHOOK_POLL_INTERVAL_MS` | não | `2000` | Manter 2000 em produção nesta fase |
| `WEBHOOK_BATCH_SIZE` | não | `20` | Inteiro entre 1 e 100 |
| `WEBHOOK_HTTP_TIMEOUT_MS` | não | `10000` | Manter 10000 em produção nesta fase |
| `WEBHOOK_PROCESSING_LEASE_MS` | não | `60000` | Deve ser maior que o timeout HTTP |

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
- Campos aditivos no payload mantêm `schema_version: 1`.
- Remoção, renomeação ou alteração semântica exige nova versão de schema e estratégia de convivência.
- O worker deve ser implantado somente após a migration e pode permanecer desabilitado enquanto a API começa a preencher a outbox.
- Rollback do worker não afeta a API; eventos permanecem persistidos.
- Rollback da API para versão que não gera eventos não remove tabelas imediatamente.

## 17. Estratégia de testes

### 17.1 Unitários

- serialização determinística e limite de 64 KB;
- assinatura HMAC sobre os bytes exatos;
- duas assinaturas durante grace period e uma após expiração;
- criptografia e descriptografia AES-GCM;
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
- lease expirado é recuperado;
- DLQ e remoção da outbox são atômicas;
- replay preserva `eventId` e audita o usuário.

### 17.3 Contrato HTTP

- CRUD autenticado para `ADMIN` e `OPERATOR`;
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
- `408`, `425`, `429` e `5xx`;
- redirect e `4xx` permanente;
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
- [ ] Processamento é serial e preserva ordem por pedido na limitação documentada.
- [ ] Cada chamada usa timeout de dez segundos.
- [ ] Falhas retentáveis seguem 1m/5m/30m/2h/12h.
- [ ] Após a tentativa inicial e cinco retries, o evento vai para a DLQ.
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
| Endpoint lento bloquear o processamento serial | Alta | Médio | Timeout de 10 segundos e retry reagendado |
| Duplicata gerar efeito repetido no cliente | Média | Alto | `X-Event-Id`, documentação e testes de idempotência |
| Vazamento de secret | Baixa | Alto | AES-GCM, redaction, exibição única, rotação e revisão de segurança |
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
