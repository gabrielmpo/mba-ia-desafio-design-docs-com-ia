# RFC — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Autor** | Gabriel Oliveira |
| **Status** | Em revisão |
| **Data** | 30/07/2026 |
| **Revisores** | Larissa (Tech Lead), Marcos (Product Manager), Bruno (Engenharia de Pedidos), Diego (Engenharia de Plataforma), Sofia (Segurança) |
| **Escopo** | Outbound webhooks para mudanças de status de pedidos |

## TL;DR

Propõe-se adicionar ao OMS um mecanismo assíncrono de outbound webhooks para notificar sistemas de clientes quando pedidos mudarem de status. A solução registra cada evento em uma outbox no MySQL, dentro da mesma transação que altera o pedido, e realiza as entregas por um worker Node.js separado que consulta eventos pendentes a cada dois segundos.

A entrega terá semântica at-least-once, identificação por UUID e retry com backoff até encaminhamento para uma dead-letter queue persistida. As requisições serão assinadas com HMAC-SHA256 usando uma secret exclusiva por endpoint, com suporte à rotação. A proposta reutiliza o Prisma, a estrutura modular, os erros, a autenticação, a validação Zod e o logger Pino existentes no OMS.

Essa abordagem atende ao objetivo de notificação em menos de dez segundos sem incluir chamadas externas na transação crítica de pedidos nem introduzir Redis ou um broker de mensagens nesta fase.

## Contexto e problema

Atlas Comercial, MaxDistribuição e Nova Cargo solicitaram notificações em tempo próximo do real quando os pedidos mudarem de status. Atualmente, esses clientes consultam periodicamente `GET /orders`, o que aumenta o custo das integrações e atrasa a percepção das mudanças. Para os clientes, uma entrega abaixo de dez segundos é aceitável.

O ponto crítico da solução é `OrderService.changeStatus`, em `src/modules/orders/order.service.ts`. O método já executa uma transação que:

- valida a máquina de estados;
- atualiza o pedido;
- registra o histórico da transição;
- debita ou repõe estoque quando aplicável.

Uma chamada HTTP síncrona nesse fluxo tornaria a mudança de status dependente da latência e da disponibilidade de um sistema externo. Se a notificação fosse registrada somente depois do commit, surgiria outra falha possível: o status poderia mudar sem que o evento correspondente fosse criado.

Além da consistência, a solução precisa lidar com endpoints temporariamente indisponíveis, duplicação de entregas, autenticação do emissor, rotação de credenciais e operação de falhas permanentes. Ela deve fazê-lo preservando os padrões da aplicação e sem adicionar infraestrutura desproporcional ao estágio atual da feature.

## Objetivos

- Notificar alterações de status normalmente em menos de dez segundos.
- Garantir que toda transição confirmada que possua assinantes gere evento persistido.
- Manter latência e falhas externas fora da transação de pedidos.
- Recuperar automaticamente falhas transitórias de entrega.
- Permitir que o consumidor valide origem e integridade da requisição.
- Preservar evidências e permitir reprocessamento controlado de falhas permanentes.
- Reutilizar a stack e os padrões operacionais existentes.

## Não objetivos

Esta proposta não inclui:

- envio de emails quando um endpoint falhar;
- dashboard visual para configuração ou acompanhamento;
- rate limiting de saída;
- webhooks recebidos de clientes;
- Redis, broker dedicado ou escala horizontal do worker;
- política definitiva de arquivamento da outbox;
- garantia exactly-once ou ordering global.

Esses itens foram descartados ou adiados durante a reunião e podem ser reavaliados com dados de produção.

## Proposta técnica

### Visão geral

O módulo de webhooks seguirá a mesma organização dos demais domínios do OMS e será responsável por configuração de endpoints, criação dos eventos, entregas, retry, histórico e DLQ. O fluxo proposto possui quatro partes:

1. **Configuração:** clientes autenticados cadastram endpoints HTTPS e os status que desejam receber. O sistema gera uma secret exclusiva por endpoint.
2. **Registro transacional:** ao mudar o status de um pedido, o OMS identifica os endpoints interessados e grava eventos na outbox usando o mesmo `Prisma.TransactionClient`.
3. **Entrega assíncrona:** um worker separado consulta a outbox por polling, assina o payload e chama os endpoints.
4. **Recuperação:** falhas transitórias são reagendadas; falhas que excedem a política de retry são preservadas na DLQ e podem ser reprocessadas por administrador.

### Registro atômico e snapshot

O evento será criado dentro da transação de `changeStatus`. Se a inserção falhar, a mudança do pedido também será revertida. O payload será renderizado e persistido naquele momento, garantindo que represente a transição original mesmo que o pedido volte a mudar antes da entrega.

Cada evento receberá um UUID imutável. O corpo será enxuto e conterá a identificação do evento, timestamp, pedido, cliente, status anterior, novo status e total do pedido. Os itens não serão incorporados; consumidores que precisarem de detalhes poderão consultar o endpoint existente de pedidos.

### Worker e latência

O worker será um processo Node.js distinto da API, com ciclo de vida e `PrismaClient` próprios. Ele consultará pequenos lotes de eventos pendentes a cada dois segundos, priorizando os mais antigos.

Inicialmente haverá apenas um worker. Isso reduz a complexidade de concorrência e preserva a ordem por pedido enquanto o processamento permanecer serial. Escala horizontal, locking e particionamento por `order_id` ficam adiados até que métricas de volume indiquem necessidade.

### Entrega, retry e DLQ

A chamada terá timeout de dez segundos. Em caso de falha de rede, timeout ou resposta considerada não bem-sucedida, o evento será reagendado conforme a sequência de backoff:

`1 minuto → 5 minutos → 30 minutos → 2 horas → 12 horas`

Após cinco retries, o evento será movido para uma tabela de dead letter com payload, motivo da falha e timestamps. Um endpoint administrativo permitirá o replay manual, exigindo role `ADMIN` e registro de auditoria.

### Segurança

Todas as URLs cadastradas deverão usar HTTPS. O worker assinará o corpo exato com HMAC-SHA256 e uma secret exclusiva do endpoint. A entrega incluirá:

- `X-Event-Id`;
- `X-Webhook-Id`;
- `X-Signature`;
- `X-Timestamp`;
- `Content-Type: application/json`.

A secret poderá ser rotacionada. Durante 24 horas, a credencial anterior continuará válida para permitir a transição do consumidor. O payload terá limite de 64 KB.

### Semântica de entrega

A garantia será at-least-once. Em falhas ambíguas, o mesmo evento poderá ser entregue novamente com o mesmo `X-Event-Id`. Os consumidores deverão implementar idempotência e deduplicação usando esse identificador.

Exactly-once não será prometido porque exigiria coordenação com cada sistema externo e aumentaria substancialmente a complexidade sem eliminar todos os efeitos duplicados.

### Compatibilidade com o OMS

A feature será integrada ao monólito e reutilizará:

- Prisma e a transação existente em `src/modules/orders/order.service.ts`;
- composição de rotas e autenticação exemplificadas em `src/modules/orders/order.routes.ts`;
- adaptação HTTP e propagação de erros de `src/modules/orders/order.controller.ts`;
- `AppError` e códigos estruturados em `src/shared/errors/index.ts`;
- validação Zod por `src/middlewares/validate.middleware.ts`;
- autenticação por `src/middlewares/auth.middleware.ts`;
- logging Pino de `src/shared/logger/index.ts`;
- padrão de entry point e shutdown de `src/server.ts`.

O detalhamento de schemas, endpoints, modelos, estados internos, códigos `WEBHOOK_*` e algoritmos de claim será definido no FDD.

## Alternativas consideradas

### Chamada síncrona no serviço de pedidos

**Vantagem:** implementação inicial simples e entrega imediata.

**Motivo do descarte:** um endpoint lento ou indisponível aumentaria a duração da transação e poderia impedir mudanças válidas de status. Também criaria uma decisão inadequada entre reverter o pedido ou perder a notificação.

### Redis Streams ou broker de mensagens

**Vantagem:** primitivas próprias para consumo, escala e retenção de mensagens.

**Motivo do descarte:** exigiria nova infraestrutura, monitoramento e conhecimento operacional. Para o volume inicial desconhecido e uma equipe pequena, a outbox no MySQL atende aos requisitos com menor custo.

### Trigger do MySQL para notificar o worker

**Vantagem:** reação imediata a alterações no banco.

**Motivo do descarte:** triggers executam SQL, mas não fornecem mecanismo adequado para despertar um processo externo. Qualquer ponte adicional seria mais frágil do que polling explícito.

### Exactly-once

**Vantagem:** esconder duplicatas dos consumidores.

**Motivo do descarte:** não é possível garantir exactly-once de ponta a ponta sem coordenação com o sistema do cliente. A complexidade não se justifica frente à alternativa at-least-once com identificador idempotente.

## Questões em aberto

As questões abaixo devem ser resolvidas ou explicitamente mantidas abertas no FDD:

1. Qual será o tamanho inicial dos lotes de polling?
2. Qual estratégia de claim e locking evitará reprocessamento concorrente em reinícios?
3. Quais códigos HTTP serão considerados retentáveis e quais serão falhas permanentes?
4. O backoff terá jitter para evitar sincronização de retries?
5. Como as secrets serão protegidas em repouso e mascaradas em logs e respostas?
6. Qual será o formato canônico de `X-Signature` e a regra exata de assinatura do corpo?
7. Como o contrato do evento será versionado sem quebrar consumidores?
8. Qual retenção será aplicada ao histórico de entregas e à DLQ?
9. Quais métricas de volume dispararão a discussão sobre rate limiting e múltiplos workers?

## Impactos

### Aplicação

- novo módulo de webhooks;
- nova entry point de worker;
- extensão transacional de `OrderService.changeStatus`;
- novos endpoints autenticados e administrativos;
- novos modelos Prisma para configuração, outbox, entregas e DLQ.

### Infraestrutura e operação

- um processo adicional usando o mesmo MySQL;
- necessidade de health check e deploy independentes para o worker;
- crescimento de tabelas operacionais;
- métricas e alertas para atraso, falhas, retries e DLQ.

### Clientes

- substituição gradual de polling por notificação;
- obrigação de validar HMAC e implementar idempotência;
- capacidade de escolher os status recebidos;
- acesso a histórico de entregas para diagnóstico.

## Riscos e mitigação

| Risco | Impacto | Mitigação proposta |
| --- | --- | --- |
| Crescimento da outbox degradar consultas | Atraso de entrega e carga no MySQL | Índices por status e data, lotes pequenos e futura política de arquivamento |
| Endpoint lento ocupar o worker | Aumento do backlog | Timeout de 10 segundos, retry reagendado e métricas de lag |
| Duplicata causar efeito repetido no cliente | Inconsistência externa | `X-Event-Id`, documentação e testes de idempotência |
| Vazamento de secret | Falsificação de mensagens | Secret exclusiva, rotação, proteção em repouso e mascaramento |
| Single worker limitar throughput | Atraso em picos | Monitorar lag; evoluir para locking/particionamento quando necessário |
| Evento permanecer irrecuperável | Perda operacional | DLQ persistida, histórico, replay admin e auditoria |

## Rollout e validação

A implementação está estimada em três sprints, incluindo pelo menos dois dias úteis para revisão de segurança. O rollout deve ocorrer gradualmente:

1. migrations e módulo de configuração;
2. geração transacional de eventos com entrega desabilitada ou controlada;
3. worker em ambiente de teste com endpoints simulados;
4. revisão de HMAC, geração e rotação de secrets;
5. piloto com cliente selecionado;
6. habilitação progressiva e acompanhamento de métricas.

Antes da liberação geral, deverão ser validados rollback transacional, retries, duplicatas, timeout, DLQ, replay administrativo, rotação de secret e comportamento sob backlog.

## Decisões relacionadas

- [ADR-001 — Outbox transacional no MySQL](adrs/ADR-001-outbox-transacional-no-mysql.md)
- [ADR-002 — Worker separado com polling](adrs/ADR-002-worker-separado-com-polling.md)
- [ADR-003 — Retry com backoff e DLQ](adrs/ADR-003-retry-com-backoff-e-dlq.md)
- [ADR-004 — Autenticação HMAC e rotação de secret](adrs/ADR-004-autenticacao-hmac-e-rotacao-de-secret.md)
- [ADR-005 — Entrega at-least-once](adrs/ADR-005-entrega-at-least-once.md)
- [ADR-006 — Snapshot e contrato do evento](adrs/ADR-006-snapshot-e-contrato-do-evento.md)
- [ADR-007 — Reuso dos padrões do OMS](adrs/ADR-007-reuso-dos-padroes-do-oms.md)

## Rastreabilidade

As evidências usadas nesta proposta estão consolidadas em [INVENTARIO-EVIDENCIAS.md](INVENTARIO-EVIDENCIAS.md), com referências à `TRANSCRICAO.md` e ao código existente. O FDD e o Tracker deverão manter os identificadores e fontes compatíveis com esse inventário.

