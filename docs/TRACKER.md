# Tracker de Rastreabilidade — Sistema de Webhooks

Este tracker liga requisitos, decisões, restrições, contexto e evidências técnicas às suas origens. Itens ainda declarados como questões abertas no RFC/FDD não são tratados aqui como decisões confirmadas.

## Cobertura

- **99 itens rastreados**: 79 originados da transcrição e 20 do código.
- **79,8% das linhas** possuem fonte `TRANSCRICAO` com timestamp e falante.
- **20 linhas** possuem fonte `CODIGO`, cobrindo 11 caminhos reais.
- A matriz cobre os identificadores consolidados no inventário (`CTX-01..14`, `RF-01..17`, `RNF-01..15`, `DEC-01..26`, `EXC-01..07` e `COD-01..20`) e, portanto, supera a cobertura mínima de 80% dos itens identificáveis do pacote.

## Matriz

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| DEC-01 | `docs/adrs/ADR-001-outbox-transacional-no-mysql.md` | Decisão | Usar Transactional Outbox no MySQL existente. | `TRANSCRICAO` | `[09:06] Diego` |
| DEC-02 | `docs/adrs/ADR-001-outbox-transacional-no-mysql.md` | Decisão | Inserir o evento na mesma transação da mudança de status. | `TRANSCRICAO` | `[09:06] Diego` |
| DEC-03 | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Decisão | Fazer polling da outbox a cada 2 segundos. | `TRANSCRICAO` | `[09:09] Diego` |
| DEC-04 | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Decisão | Executar o worker em processo separado da API. | `TRANSCRICAO` | `[09:11] Diego` |
| DEC-05 | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Decisão | Criar entry point `src/worker.ts`. | `TRANSCRICAO` | `[09:11] Larissa` |
| DEC-06 | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Decisão | Usar uma instância Prisma por processo. | `TRANSCRICAO` | `[09:29] Diego` |
| DEC-07 | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Decisão | Operar inicialmente com single worker. | `TRANSCRICAO` | `[09:12] Diego` |
| DEC-08 | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Decisão | Ordenar pendentes por `created_at`. | `TRANSCRICAO` | `[09:08] Diego` |
| DEC-09 | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Decisão | Aceitar ordering por pedido apenas enquanto houver single worker. | `TRANSCRICAO` | `[09:12] Diego` |
| DEC-10 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Decisão | Aplicar a política registrada como 5 tentativas; a relação entre chamada inicial e retries permanece em aberto. | `TRANSCRICAO` | `[09:15] Diego` |
| DEC-11 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Decisão | Usar backoff 1m/5m/30m/2h/12h. | `TRANSCRICAO` | `[09:17] Diego` |
| DEC-12 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Decisão | Persistir DLQ em tabela separada. | `TRANSCRICAO` | `[09:17] Diego` |
| DEC-13 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Decisão | Permitir replay manual da DLQ por endpoint admin. | `TRANSCRICAO` | `[09:18] Diego` |
| DEC-14 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Decisão | Exigir role ADMIN e auditoria no replay. | `TRANSCRICAO` | `[09:35] Larissa` |
| DEC-15 | `docs/adrs/ADR-004-autenticacao-hmac-e-rotacao-de-secret.md` | Decisão | Assinar o corpo com HMAC-SHA256. | `TRANSCRICAO` | `[09:19] Sofia` |
| DEC-16 | `docs/adrs/ADR-004-autenticacao-hmac-e-rotacao-de-secret.md` | Decisão | Manter uma secret exclusiva por endpoint. | `TRANSCRICAO` | `[09:21] Sofia` |
| DEC-17 | `docs/adrs/ADR-004-autenticacao-hmac-e-rotacao-de-secret.md` | Decisão | Permitir rotação com grace period de 24 horas. | `TRANSCRICAO` | `[09:21] Sofia` |
| DEC-18 | `docs/adrs/ADR-004-autenticacao-hmac-e-rotacao-de-secret.md` | Decisão | Exigir HTTPS no endpoint cadastrado. | `TRANSCRICAO` | `[09:23] Sofia` |
| DEC-19 | `docs/adrs/ADR-004-autenticacao-hmac-e-rotacao-de-secret.md` | Decisão | Limitar payload a 64 KB e falhar ao exceder. | `TRANSCRICAO` | `[09:23] Sofia` |
| DEC-20 | `docs/adrs/ADR-005-entrega-at-least-once.md` | Decisão | Garantir entrega at-least-once. | `TRANSCRICAO` | `[09:24] Diego` |
| DEC-21 | `docs/adrs/ADR-005-entrega-at-least-once.md` | Decisão | Usar UUID em `X-Event-Id` para deduplicação. | `TRANSCRICAO` | `[09:25] Diego` |
| DEC-22 | `docs/FDD.md` | Decisão | Filtrar status interessados antes de inserir na outbox. | `TRANSCRICAO` | `[09:33] Bruno` |
| DEC-23 | `docs/FDD.md` | Decisão | Aplicar timeout HTTP de 10 segundos. | `TRANSCRICAO` | `[09:42] Diego` |
| DEC-24 | `docs/adrs/ADR-007-reuso-dos-padroes-do-oms.md` | Decisão | Reutilizar estrutura modular, erros, Zod, Pino e middlewares existentes. | `TRANSCRICAO` | `[09:27] Bruno` |
| DEC-25 | `docs/adrs/ADR-006-snapshot-e-contrato-do-evento.md` | Decisão | Usar UUID nos registros, seguindo padrão do projeto. | `TRANSCRICAO` | `[09:51] Larissa` |
| DEC-26 | `docs/adrs/ADR-006-snapshot-e-contrato-do-evento.md` | Decisão | Persistir payload em snapshot na inserção. | `TRANSCRICAO` | `[09:51] Larissa` |
| RF-01 | `docs/PRD.md` | Requisito Funcional | Notificar alterações de status de pedido por outbound webhook. | `TRANSCRICAO` | `[09:00] Marcos` |
| RF-02 | `docs/PRD.md` | Requisito Funcional | Cadastrar endpoint por POST. | `TRANSCRICAO` | `[09:31] Marcos` |
| RF-03 | `docs/PRD.md` | Requisito Funcional | Gerar e devolver secret na criação. | `TRANSCRICAO` | `[09:31] Marcos` |
| RF-04 | `docs/PRD.md` | Requisito Funcional | Associar webhook a um customer informado na requisição. | `TRANSCRICAO` | `[09:31] Marcos` |
| RF-05 | `docs/PRD.md` | Requisito Funcional | Configurar lista de status por endpoint. | `TRANSCRICAO` | `[09:31] Marcos` |
| RF-06 | `docs/PRD.md` | Requisito Funcional | Editar configuração por PATCH. | `TRANSCRICAO` | `[09:33] Bruno` |
| RF-07 | `docs/PRD.md` | Requisito Funcional | Remover configuração por DELETE. | `TRANSCRICAO` | `[09:33] Bruno` |
| RF-08 | `docs/PRD.md` | Requisito Funcional | Listar webhooks de um customer por GET. | `TRANSCRICAO` | `[09:33] Bruno` |
| RF-09 | `docs/PRD.md` | Requisito Funcional | Listar histórico de entregas por endpoint. | `TRANSCRICAO` | `[09:34] Marcos` |
| RF-10 | `docs/PRD.md` | Requisito Funcional | Mostrar sucesso/falha, payload, resposta e duração no histórico. | `TRANSCRICAO` | `[09:34] Marcos` |
| RF-11 | `docs/PRD.md` | Requisito Funcional | Reprocessar item da DLQ manualmente. | `TRANSCRICAO` | `[09:18] Diego` |
| RF-12 | `docs/PRD.md` | Requisito Funcional | Auditar usuário que solicitou replay. | `TRANSCRICAO` | `[09:35] Larissa` |
| RF-13 | `docs/PRD.md` | Requisito Funcional | Rotacionar secret pela API. | `TRANSCRICAO` | `[09:21] Sofia` |
| RF-14 | `docs/PRD.md` | Requisito Funcional | Entregar `order.status_changed`. | `TRANSCRICAO` | `[09:43] Diego` |
| RF-15 | `docs/PRD.md` | Requisito Funcional | Enviar identificadores de evento e webhook nos headers. | `TRANSCRICAO` | `[09:44] Diego` |
| RF-16 | `docs/PRD.md` | Requisito Funcional | Enviar assinatura e timestamp nos headers. | `TRANSCRICAO` | `[09:44] Diego` |
| RF-17 | `docs/PRD.md` | Requisito Funcional | Permitir consultar detalhes do pedido pela API existente. | `TRANSCRICAO` | `[09:43] Diego` |
| RNF-01 | `docs/PRD.md` | Requisito Não Funcional | Entregar normalmente em menos de 10 segundos. | `TRANSCRICAO` | `[09:01] Bruno` |
| RNF-02 | `docs/PRD.md` | Requisito Não Funcional | Não bloquear a mudança de status com chamada externa. | `TRANSCRICAO` | `[09:03] Larissa` |
| RNF-03 | `docs/PRD.md` | Requisito Não Funcional | Garantir atomicidade entre status e evento. | `TRANSCRICAO` | `[09:06] Diego` |
| RNF-04 | `docs/PRD.md` | Requisito Não Funcional | Suportar indisponibilidade temporária do consumidor. | `TRANSCRICAO` | `[09:14] Larissa` |
| RNF-05 | `docs/PRD.md` | Requisito Não Funcional | Não manter retries indefinidamente. | `TRANSCRICAO` | `[09:15] Diego` |
| RNF-06 | `docs/PRD.md` | Requisito Não Funcional | Preservar evidências de falha para debug. | `TRANSCRICAO` | `[09:18] Diego` |
| RNF-07 | `docs/PRD.md` | Requisito Não Funcional | Autenticar origem e integridade do payload. | `TRANSCRICAO` | `[09:19] Sofia` |
| RNF-08 | `docs/PRD.md` | Requisito Não Funcional | Isolar comprometimento de secrets por endpoint. | `TRANSCRICAO` | `[09:21] Sofia` |
| RNF-09 | `docs/PRD.md` | Requisito Não Funcional | Usar somente TLS/HTTPS. | `TRANSCRICAO` | `[09:23] Sofia` |
| RNF-10 | `docs/PRD.md` | Requisito Não Funcional | Impor tamanho máximo de payload de 64 KB. | `TRANSCRICAO` | `[09:23] Sofia` |
| RNF-11 | `docs/PRD.md` | Requisito Não Funcional | Tolerar duplicação por semântica at-least-once. | `TRANSCRICAO` | `[09:24] Diego` |
| RNF-12 | `docs/PRD.md` | Requisito Não Funcional | Manter payload enxuto, sem itens. | `TRANSCRICAO` | `[09:43] Diego` |
| RNF-13 | `docs/PRD.md` | Requisito Não Funcional | Aplicar timeout de 10 segundos. | `TRANSCRICAO` | `[09:42] Diego` |
| RNF-14 | `docs/PRD.md` | Requisito Não Funcional | Usar logging Pino e códigos `WEBHOOK_*`. | `TRANSCRICAO` | `[09:28] Bruno` |
| RNF-15 | `docs/PRD.md` | Requisito Não Funcional | Concluir em três sprints incluindo revisão de segurança. | `TRANSCRICAO` | `[09:45] Marcos` |
| EXC-01 | `docs/PRD.md` | Fora de escopo | Arquivar eventos entregues após 30 dias. | `TRANSCRICAO` | `[09:08] Diego` |
| EXC-02 | `docs/PRD.md` | Fora de escopo | Escala horizontal e particionamento por `order_id`. | `TRANSCRICAO` | `[09:12] Diego` |
| EXC-03 | `docs/PRD.md` | Fora de escopo | Email de alerta/fallback. | `TRANSCRICAO` | `[09:37] Larissa` |
| EXC-04 | `docs/PRD.md` | Fora de escopo | Rate limiting de saída. | `TRANSCRICAO` | `[09:38] Diego` |
| EXC-05 | `docs/PRD.md` | Fora de escopo | Dashboard visual. | `TRANSCRICAO` | `[09:39] Larissa` |
| EXC-06 | `docs/PRD.md` | Fora de escopo | Webhooks inbound. | `TRANSCRICAO` | `[09:02] Marcos` |
| EXC-07 | `docs/PRD.md` | Fora de escopo | Enviar itens completos no payload. | `TRANSCRICAO` | `[09:43] Diego` |
| COD-01 | `docs/FDD.md` | Evidência de código | `changeStatus` já usa `$transaction`. | `CODIGO` | `src/modules/orders/order.service.ts` |
| COD-02 | `docs/FDD.md` | Evidência de código | A transação lê pedido e itens. | `CODIGO` | `src/modules/orders/order.service.ts` |
| COD-03 | `docs/FDD.md` | Evidência de código | A máquina de estados usa `canTransition`. | `CODIGO` | `src/modules/orders/order.status.ts` |
| COD-04 | `docs/FDD.md` | Evidência de código | Débito de estoque ocorre na transação. | `CODIGO` | `src/modules/orders/order.service.ts` |
| COD-05 | `docs/FDD.md` | Evidência de código | Reposição de estoque ocorre na transação. | `CODIGO` | `src/modules/orders/order.service.ts` |
| COD-06 | `docs/FDD.md` | Evidência de código | Histórico é criado por `orderStatusHistory`. | `CODIGO` | `src/modules/orders/order.service.ts` |
| COD-07 | `docs/FDD.md` | Evidência de código | Serviço recebe `PrismaClient`. | `CODIGO` | `src/modules/orders/order.service.ts` |
| COD-08 | `docs/FDD.md` | Evidência de código | Há tipo `Prisma.TransactionClient`. | `CODIGO` | `src/modules/orders/order.service.ts` |
| COD-09 | `docs/FDD.md` | Evidência de código | Controller encaminha erros por `next(err)`. | `CODIGO` | `src/modules/orders/order.controller.ts` |
| COD-10 | `docs/FDD.md` | Evidência de código | Controller exige `req.user` nas mutações. | `CODIGO` | `src/modules/orders/order.controller.ts` |
| COD-11 | `docs/FDD.md` | Evidência de código | Rotas aplicam `authenticate`. | `CODIGO` | `src/modules/orders/order.routes.ts` |
| COD-12 | `docs/FDD.md` | Evidência de código | Rotas usam middleware `validate`. | `CODIGO` | `src/modules/orders/order.routes.ts` |
| COD-13 | `docs/FDD.md` | Evidência de código | Schemas de pedidos ficam no módulo. | `CODIGO` | `src/modules/orders/order.schemas.ts` |
| COD-14 | `docs/FDD.md` | Evidência de código | Repositório é abstraído por interface. | `CODIGO` | `src/modules/orders/order.repository.ts` |
| COD-15 | `docs/FDD.md` | Evidência de código | Erros são exportados centralmente. | `CODIGO` | `src/shared/errors/index.ts` |
| COD-16 | `docs/FDD.md` | Evidência de código | `InvalidStatusTransitionError` já existe. | `CODIGO` | `src/shared/errors/index.ts` |
| COD-17 | `docs/FDD.md` | Evidência de código | Respostas paginadas usam helper compartilhado. | `CODIGO` | `src/shared/http/response.ts` |
| COD-18 | `docs/FDD.md` | Evidência de código | API inicia em entry point próprio. | `CODIGO` | `src/server.ts` |
| COD-19 | `docs/FDD.md` | Evidência de código | Banco é importado de configuração compartilhada. | `CODIGO` | `src/config/database.ts` |
| COD-20 | `docs/FDD.md` | Evidência de código | Logger Pino é compartilhado. | `CODIGO` | `src/shared/logger/index.ts` |
| CTX-01 | `docs/PRD.md` | Contexto | Três clientes B2B solicitaram a feature. | `TRANSCRICAO` | `[09:00] Marcos` |
| CTX-02 | `docs/PRD.md` | Contexto | Atlas sinalizou risco de migração. | `TRANSCRICAO` | `[09:00] Marcos` |
| CTX-03 | `docs/PRD.md` | Contexto | Clientes atualmente fazem polling de `GET /orders`. | `TRANSCRICAO` | `[09:00] Marcos` |
| CTX-04 | `docs/PRD.md` | Contexto | A solução é somente outbound. | `TRANSCRICAO` | `[09:02] Marcos` |
| CTX-05 | `docs/PRD.md` | Contexto | A transação de status já é pesada. | `TRANSCRICAO` | `[09:04] Bruno` |
| CTX-06 | `docs/PRD.md` | Contexto | A equipe é pequena e evita nova infraestrutura. | `TRANSCRICAO` | `[09:07] Diego` |
| CTX-07 | `docs/PRD.md` | Contexto | Índices necessários: status e `created_at`. | `TRANSCRICAO` | `[09:08] Diego` |
| CTX-08 | `docs/PRD.md` | Contexto | Single worker é limitação conhecida. | `TRANSCRICAO` | `[09:12] Diego` |
| CTX-09 | `docs/PRD.md` | Contexto | Clientes já tiveram manutenção de duas horas. | `TRANSCRICAO` | `[09:16] Diego` |
| CTX-10 | `docs/PRD.md` | Contexto | A secret pode vazar em logs de clientes. | `TRANSCRICAO` | `[09:22] Diego` |
| CTX-11 | `docs/PRD.md` | Contexto | Consumidor precisa implementar deduplicação. | `TRANSCRICAO` | `[09:25] Diego` |
| CTX-12 | `docs/PRD.md` | Contexto | CRUD usa usuários autenticados que representam clientes. | `TRANSCRICAO` | `[09:31] Marcos` |
| CTX-13 | `docs/PRD.md` | Contexto | Sofia requer dois dias úteis de revisão de segurança. | `TRANSCRICAO` | `[09:46] Sofia` |
| CTX-14 | `docs/PRD.md` | Contexto | Prazo estimado é de três sprints. | `TRANSCRICAO` | `[09:45] Marcos` |

## Notas de integridade

- Uma mesma evidência pode sustentar conteúdo em mais de um documento; a coluna **Documento** aponta para o artefato principal da decisão ou requisito.
- As propostas não normativas e questões abertas permanecem identificadas como tal no RFC e no FDD; elas não ganham status de requisito ou decisão por aparecerem neste pacote.
- O inventário detalhado de evidências permanece em [`docs/INVENTARIO-EVIDENCIAS.md`](INVENTARIO-EVIDENCIAS.md).
