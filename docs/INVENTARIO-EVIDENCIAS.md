# Inventário de evidências

Este inventário é a base de rastreabilidade para os design docs do Sistema de Webhooks de Notificação de Pedidos. Ele separa decisões, requisitos, alternativas, exclusões, questões abertas, referências de código e contexto. Os itens são fatos extraídos da reunião ou do código; hipóteses de implementação devem permanecer identificadas como questões abertas até decisão posterior.

## Resumo

| Categoria | Quantidade |
| --- | ---: |
| Decisões | 26 |
| Requisitos funcionais | 17 |
| Requisitos não funcionais | 15 |
| Alternativas consideradas | 9 |
| Exclusões e adiamentos | 7 |
| Questões abertas | 14 |
| Referências de código | 20 |
| Contexto, riscos e dependências | 14 |
| **Total** | **122** |

## Decisões (26)

| ID | Evidência | Fonte |
| --- | --- | --- |
| DEC-01 | Usar Transactional Outbox no MySQL existente. | `[09:06] Diego`–`[09:08] Larissa` |
| DEC-02 | Inserir o evento na mesma transação da mudança de status. | `[09:06] Diego`; `[09:40] Bruno` |
| DEC-03 | Fazer polling da outbox a cada 2 segundos. | `[09:09] Diego`–`[09:10] Larissa` |
| DEC-04 | Executar o worker em processo separado da API. | `[09:11] Diego` |
| DEC-05 | Criar entry point `src/worker.ts`. | `[09:11] Larissa` |
| DEC-06 | Usar uma instância Prisma por processo. | `[09:29] Diego`–`[09:30] Bruno` |
| DEC-07 | Operar inicialmente com single worker. | `[09:12] Diego`–`[09:13] Larissa` |
| DEC-08 | Ordenar pendentes por `created_at`. | `[09:08] Diego`; `[09:12] Diego` |
| DEC-09 | Aceitar ordering por pedido apenas enquanto houver single worker. | `[09:12]`–`[09:14]` |
| DEC-10 | Fazer 5 retries. | `[09:15]`–`[09:17]` |
| DEC-11 | Usar backoff 1m/5m/30m/2h/12h. | `[09:17] Diego` |
| DEC-12 | Persistir DLQ em tabela separada. | `[09:17]`–`[09:18]` |
| DEC-13 | Permitir replay manual da DLQ por endpoint admin. | `[09:18] Diego` |
| DEC-14 | Exigir role ADMIN e auditoria no replay. | `[09:35]`–`[09:36]` |
| DEC-15 | Assinar o corpo com HMAC-SHA256. | `[09:19]`–`[09:22]` |
| DEC-16 | Manter uma secret exclusiva por endpoint. | `[09:21] Sofia` |
| DEC-17 | Permitir rotação com grace period de 24 horas. | `[09:21]`–`[09:22]` |
| DEC-18 | Exigir HTTPS no endpoint cadastrado. | `[09:23] Sofia` |
| DEC-19 | Limitar payload a 64 KB e falhar ao exceder. | `[09:23]`–`[09:24]` |
| DEC-20 | Garantir entrega at-least-once. | `[09:24]`–`[09:26]` |
| DEC-21 | Usar UUID em `X-Event-Id` para deduplicação. | `[09:25] Diego` |
| DEC-22 | Filtrar status interessados antes de inserir na outbox. | `[09:33]`–`[09:34]` |
| DEC-23 | Aplicar timeout HTTP de 10 segundos. | `[09:42] Diego` |
| DEC-24 | Reutilizar estrutura modular, erros, Zod, Pino e middlewares existentes. | `[09:27]`–`[09:30]` |
| DEC-25 | Usar UUID nos registros, seguindo padrão do projeto. | `[09:51] Larissa` |
| DEC-26 | Persistir payload em snapshot na inserção. | `[09:51]`–`[09:52]` |

## Requisitos funcionais (17)

| ID | Evidência | Fonte |
| --- | --- | --- |
| RF-01 | Notificar alterações de status de pedido por outbound webhook. | `[09:00]`–`[09:03]` |
| RF-02 | Cadastrar endpoint por POST. | `[09:31] Marcos` |
| RF-03 | Gerar e devolver secret na criação. | `[09:31] Marcos` |
| RF-04 | Associar webhook a um customer informado na requisição. | `[09:31]`–`[09:33]` |
| RF-05 | Configurar lista de status por endpoint. | `[09:31]`–`[09:34]` |
| RF-06 | Editar configuração por PATCH. | `[09:33] Bruno` |
| RF-07 | Remover configuração por DELETE. | `[09:33] Bruno` |
| RF-08 | Listar webhooks de um customer por GET. | `[09:33] Bruno` |
| RF-09 | Listar histórico de entregas por endpoint. | `[09:34] Marcos` |
| RF-10 | Mostrar sucesso/falha, payload, resposta e duração no histórico. | `[09:34] Marcos` |
| RF-11 | Reprocessar item da DLQ manualmente. | `[09:18] Diego` |
| RF-12 | Auditar usuário que solicitou replay. | `[09:35]`–`[09:36]` |
| RF-13 | Rotacionar secret pela API. | `[09:21] Sofia` |
| RF-14 | Entregar `order.status_changed`. | `[09:43] Diego` |
| RF-15 | Enviar identificadores de evento e webhook nos headers. | `[09:44]`–`[09:45]` |
| RF-16 | Enviar assinatura e timestamp nos headers. | `[09:44] Diego` |
| RF-17 | Permitir consultar detalhes do pedido pela API existente. | `[09:43]`–`[09:44]` |

## Requisitos não funcionais (15)

| ID | Evidência | Fonte |
| --- | --- | --- |
| RNF-01 | Entregar normalmente em menos de 10 segundos. | `[09:01]`–`[09:02]` |
| RNF-02 | Não bloquear a mudança de status com chamada externa. | `[09:03]`–`[09:06]` |
| RNF-03 | Garantir atomicidade entre status e evento. | `[09:06]`; `[09:40]`–`[09:41]` |
| RNF-04 | Suportar indisponibilidade temporária do consumidor. | `[09:14]`–`[09:17]` |
| RNF-05 | Não manter retries indefinidamente. | `[09:15] Diego` |
| RNF-06 | Preservar evidências de falha para debug. | `[09:18] Diego` |
| RNF-07 | Autenticar origem e integridade do payload. | `[09:19]`–`[09:20]` |
| RNF-08 | Isolar comprometimento de secrets por endpoint. | `[09:21] Sofia` |
| RNF-09 | Usar somente TLS/HTTPS. | `[09:23] Sofia` |
| RNF-10 | Impor tamanho máximo de payload de 64 KB. | `[09:23]`–`[09:24]` |
| RNF-11 | Tolerar duplicação por semântica at-least-once. | `[09:24]`–`[09:26]` |
| RNF-12 | Manter payload enxuto, sem itens. | `[09:43]`–`[09:44]` |
| RNF-13 | Aplicar timeout de 10 segundos. | `[09:42] Diego` |
| RNF-14 | Usar logging Pino e códigos `WEBHOOK_*`. | `[09:28]`–`[09:30]` |
| RNF-15 | Concluir em três sprints incluindo revisão de segurança. | `[09:45]`–`[09:47]` |

## Alternativas consideradas (9)

| ID | Alternativa | Resultado e fonte |
| --- | --- | --- |
| ALT-01 | Disparo HTTP síncrono em `changeStatus`. | Descartado; `[09:03]`–`[09:06]` |
| ALT-02 | Redis Streams/broker. | Descartado por nova infraestrutura; `[09:07]` |
| ALT-03 | Trigger MySQL para acordar o worker. | Descartado; `[09:09]` |
| ALT-04 | Worker embutido na API. | Descartado; `[09:11]` |
| ALT-05 | Múltiplos workers já na primeira fase. | Adiado; `[09:12]`–`[09:13]` |
| ALT-06 | Somente 3 retries. | Descartado; `[09:15]`–`[09:16]` |
| ALT-07 | Retry indefinido. | Descartado; `[09:15]` |
| ALT-08 | Falha permanente na própria outbox. | Preferida DLQ separada; `[09:17]`–`[09:18]` |
| ALT-09 | Exactly-once. | Descartado por complexidade; `[09:24]`–`[09:26]` |

## Exclusões e adiamentos (7)

| ID | Item fora de escopo | Fonte |
| --- | --- | --- |
| EXC-01 | Arquivar eventos entregues após 30 dias. | `[09:08] Diego` |
| EXC-02 | Escala horizontal e particionamento por `order_id`. | `[09:12]`–`[09:13]` |
| EXC-03 | Email de alerta/fallback. | `[09:37]`–`[09:38]` |
| EXC-04 | Rate limiting de saída. | `[09:38]`–`[09:39]` |
| EXC-05 | Dashboard visual. | `[09:39]`–`[09:40]` |
| EXC-06 | Webhooks inbound. | `[09:02]`–`[09:03]` |
| EXC-07 | Enviar itens completos no payload. | `[09:43]`–`[09:44]` |

## Questões abertas (14)

| ID | Questão | Origem |
| --- | --- | --- |
| QA-01 | Qual tamanho do lote de polling? | Derivada de `[09:08] Diego` |
| QA-02 | Como realizar claim/locking de eventos? | Limitação de `[09:12]`–`[09:13]` |
| QA-03 | Quais códigos HTTP são retentáveis? | Retry definido em `[09:14]`–`[09:18]` |
| QA-04 | Como aplicar jitter ao backoff? | Backoff definido em `[09:17]` |
| QA-05 | Qual política futura de retenção da DLQ? | DLQ definida em `[09:18]` |
| QA-06 | Como criptografar secrets em repouso? | Secret definida em `[09:21]` |
| QA-07 | A rotação aceita assinatura com qualquer uma das duas secrets? | Grace period em `[09:21]` |
| QA-08 | Qual formato exato de `X-Signature`? | Header definido em `[09:20]` e `[09:44]` |
| QA-09 | Qual regra de canonicalização do JSON para HMAC? | HMAC sobre o corpo em `[09:22]` |
| QA-10 | Como versionar o contrato de evento? | Payload em `[09:43]` |
| QA-11 | Qual paginação do histórico de deliveries? | Histórico em `[09:34]` |
| QA-12 | Quais roles podem administrar CRUD além de autenticação? | `[09:36]`–`[09:37]` |
| QA-13 | Quando introduzir rate limiting? | `[09:38]`–`[09:39]` |
| QA-14 | Qual volume e SLO após o lançamento? | Motivação e medição mencionadas em `[09:37]`–`[09:39]` |

## Referências de código (20)

| ID | Evidência no código | Caminho |
| --- | --- | --- |
| COD-01 | `changeStatus` já usa `$transaction`. | `src/modules/orders/order.service.ts` |
| COD-02 | A transação lê pedido e itens. | `src/modules/orders/order.service.ts` |
| COD-03 | A máquina de estados usa `canTransition`. | `src/modules/orders/order.status.ts` |
| COD-04 | Débito de estoque ocorre na transação. | `src/modules/orders/order.service.ts` |
| COD-05 | Reposição de estoque ocorre na transação. | `src/modules/orders/order.service.ts` |
| COD-06 | Histórico é criado por `orderStatusHistory`. | `src/modules/orders/order.service.ts` |
| COD-07 | Serviço recebe `PrismaClient`. | `src/modules/orders/order.service.ts` |
| COD-08 | Há tipo `Prisma.TransactionClient`. | `src/modules/orders/order.service.ts` |
| COD-09 | Controller encaminha erros por `next(err)`. | `src/modules/orders/order.controller.ts` |
| COD-10 | Controller exige `req.user` nas mutações. | `src/modules/orders/order.controller.ts` |
| COD-11 | Rotas aplicam `authenticate`. | `src/modules/orders/order.routes.ts` |
| COD-12 | Rotas usam middleware `validate`. | `src/modules/orders/order.routes.ts` |
| COD-13 | Schemas de pedidos ficam no módulo. | `src/modules/orders/order.schemas.ts` |
| COD-14 | Repositório é abstraído por interface. | `src/modules/orders/order.repository.ts` |
| COD-15 | Erros são exportados centralmente. | `src/shared/errors/index.ts` |
| COD-16 | `InvalidStatusTransitionError` já existe. | `src/shared/errors/index.ts` |
| COD-17 | Respostas paginadas usam helper compartilhado. | `src/shared/http/response.ts` |
| COD-18 | API inicia em entry point próprio. | `src/server.ts` |
| COD-19 | Banco é importado de configuração compartilhada. | `src/config/database.ts` |
| COD-20 | Logger Pino é compartilhado. | `src/shared/logger/index.ts` |

## Contexto, riscos e dependências (14)

| ID | Item | Fonte |
| --- | --- | --- |
| CTX-01 | Três clientes B2B solicitaram a feature. | `[09:00] Marcos` |
| CTX-02 | Atlas sinalizou risco de migração. | `[09:00] Marcos` |
| CTX-03 | Clientes atualmente fazem polling de `GET /orders`. | `[09:00] Marcos` |
| CTX-04 | A solução é somente outbound. | `[09:02]`–`[09:03]` |
| CTX-05 | A transação de status já é pesada. | `[09:04] Bruno` |
| CTX-06 | A equipe é pequena e evita nova infraestrutura. | `[09:07] Diego` |
| CTX-07 | Índices necessários: status e `created_at`. | `[09:08] Diego` |
| CTX-08 | Single worker é limitação conhecida. | `[09:12]`–`[09:13]` |
| CTX-09 | Clientes já tiveram manutenção de duas horas. | `[09:16] Diego` |
| CTX-10 | A secret pode vazar em logs de clientes. | `[09:22] Diego` |
| CTX-11 | Consumidor precisa implementar deduplicação. | `[09:25]`–`[09:26]` |
| CTX-12 | CRUD usa usuários autenticados que representam clientes. | `[09:31]`–`[09:33]` |
| CTX-13 | Sofia requer dois dias úteis de revisão de segurança. | `[09:46] Sofia` |
| CTX-14 | Prazo estimado é de três sprints. | `[09:45]`–`[09:47]` |

