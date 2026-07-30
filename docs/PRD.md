# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Produto** | Order Management System (OMS) |
| **Feature** | Webhooks outbound para mudanças de status de pedidos |
| **Autor** | Gabriel Oliveira |
| **Status** | Em revisão |
| **Data** | 30/07/2026 |
| **Público inicial** | Clientes B2B integrados ao OMS |

## 1. Resumo e contexto

O OMS passará a notificar sistemas de clientes B2B quando o status de um pedido mudar. A primeira fase oferece webhooks exclusivamente de saída: o OMS envia eventos aos endpoints HTTPS cadastrados pelos clientes, sem receber eventos externos.

Hoje, Atlas Comercial, MaxDistribuição e Nova Cargo consultam periodicamente `GET /orders` para descobrir mudanças. Esse polling torna as integrações mais lentas e caras e cria risco comercial: a Atlas indicou que pode migrar para um concorrente se a capacidade não for entregue até o fim do trimestre.

O produto deverá disponibilizar configuração de endpoints por cliente, filtros de status, histórico de entregas, rotação de secret e reprocessamento administrativo. A entrega será assíncrona e desacoplada da mudança de status do pedido, preservando a operação principal mesmo quando um consumidor estiver lento ou indisponível.

**Fontes:** `TRANSCRICAO.md` `[09:00] Marcos`–`[09:03] Sofia`; `[09:31] Marcos`–`[09:40] Larissa`.

## 2. Problema e motivação

### 2.1 Problema atual

- Os clientes precisam consultar repetidamente a API para identificar alterações de pedidos.
- O polling introduz atraso e custo desnecessário nas integrações.
- Não existe no OMS um mecanismo de notificação externa, eventos, filas ou webhooks.
- Uma chamada HTTP síncrona durante a mudança de status aumentaria a latência e poderia fazer a operação depender da disponibilidade do cliente.

### 2.2 Motivação

- Atender uma solicitação formal de três clientes B2B.
- Entregar notificações percebidas como em tempo real, definidas pelos clientes como recebimento em menos de 10 segundos.
- Reduzir a necessidade de polling de `GET /orders`.
- Evitar risco de perda da Atlas Comercial para um concorrente.
- Criar uma integração confiável sem bloquear nem reverter mudanças de status por falhas externas.

**Fontes:** `TRANSCRICAO.md` `[09:00] Marcos`–`[09:06] Diego`; `README.md`, seção “Contexto”.

## 3. Público-alvo e cenários de uso

### 3.1 Público-alvo

- **Clientes B2B integrados ao OMS**, representados inicialmente por Atlas Comercial, MaxDistribuição e Nova Cargo.
- **Usuários autenticados do OMS que representam clientes**, responsáveis por administrar configurações de webhook.
- **Administradores do OMS**, responsáveis por investigar falhas e reprocessar itens da DLQ.
- **Equipes técnicas dos clientes**, responsáveis por validar assinaturas, consumir eventos e deduplicar entregas.

### 3.2 Cenários de uso

1. Um usuário autenticado cadastra um endpoint HTTPS para determinado cliente e escolhe quais status deseja receber.
2. O status de um pedido muda e o OMS registra o evento de forma atômica com a alteração.
3. O OMS entrega um evento `order.status_changed` ao endpoint interessado em menos de 10 segundos durante a operação normal.
4. O consumidor valida a assinatura HMAC, processa o evento e usa `X-Event-Id` para ignorar duplicatas.
5. Uma entrega falha temporariamente e o OMS realiza novas tentativas conforme a política definida.
6. Depois do esgotamento das tentativas, o evento fica disponível na DLQ para análise.
7. Um administrador reprocessa manualmente um item da DLQ, com registro de quem executou a ação.
8. Um cliente consulta o histórico de entregas para investigar sucessos, falhas, payloads, respostas e tempos de resposta.
9. Um cliente rotaciona a secret e migra sua validação durante o período de convivência de 24 horas.

**Fontes:** `TRANSCRICAO.md` `[09:14]`–`[09:26]`; `[09:31]`–`[09:45]`.

## 4. Objetivos e métricas de sucesso

| ID | Objetivo | Métrica e meta |
| --- | --- | --- |
| OBJ-01 | Notificar mudanças de status em tempo percebido como real pelos clientes. | Latência normal entre mudança de status e primeira tentativa de entrega **inferior a 10 segundos**. |
| OBJ-02 | Desacoplar integrações externas da operação de pedidos. | **Nenhuma chamada HTTP externa** executada dentro da transação de mudança de status. |
| OBJ-03 | Evitar mudança de status confirmada sem o evento correspondente. | Registro do evento e mudança de status na **mesma transação**, de modo que ambos confirmem ou ambos sofram rollback. |
| OBJ-04 | Permitir recuperação de indisponibilidades temporárias do consumidor. | Aplicar a política de **5 tentativas**, com intervalos de `1m/5m/30m/2h/12h`, antes da DLQ. |
| OBJ-05 | Proteger autenticidade e integridade das notificações. | **100% das entregas** assinadas com HMAC-SHA256 e realizadas somente para URLs HTTPS. |
| OBJ-06 | Permitir operação e diagnóstico da feature. | Disponibilizar histórico por webhook e replay administrativo auditado da DLQ. |

As metas expressam os requisitos e garantias decididos na reunião. A reunião não definiu percentis de latência, volume esperado, disponibilidade percentual nem SLO pós-lançamento; esses indicadores permanecem pendentes de medição e decisão.

**Fontes:** `TRANSCRICAO.md` `[09:01]`–`[09:02]`; `[09:03]`–`[09:10]`; `[09:15]`–`[09:24]`; `[09:34]`–`[09:36]`.

## 5. Escopo

### 5.1 Incluído

- Webhooks outbound para mudanças de status de pedidos.
- CRUD autenticado das configurações de webhook.
- Associação da configuração a um cliente informado na requisição.
- Filtro de eventos por lista de status.
- Geração de secret exclusiva por endpoint.
- Rotação de secret via API com período de convivência de 24 horas.
- Entregas assíncronas com semântica at-least-once.
- Identificação de evento para deduplicação pelo consumidor.
- Assinatura HMAC-SHA256 e exigência de HTTPS.
- Retry com backoff e DLQ persistida.
- Histórico de entregas por webhook.
- Replay manual da DLQ restrito a `ADMIN` e auditado.
- Payload enxuto de `order.status_changed`, sem os itens do pedido.

### 5.2 Fora de escopo

| Item | Motivo/status |
| --- | --- |
| Webhooks inbound | A primeira fase cobre somente notificações enviadas pelo OMS. |
| Disparo de email após falhas | Adiado para uma fase futura, após medição de impacto. |
| Dashboard visual | Projeto separado para o time de frontend; a fase atual entrega somente endpoints. |
| Rate limiting de saída | Será observado e decidido depois se o volume tornar necessário. |
| Escala horizontal e múltiplos workers | Adiados; a primeira fase opera com um único worker. |
| Arquivamento de entregas após 30 dias | Mencionado como necessidade posterior, fora desta feature. |
| Itens completos do pedido no payload | Excluídos para manter o evento enxuto; detalhes continuam disponíveis na API de pedidos. |
| Garantia exactly-once | Descartada pela complexidade de coordenação entre OMS e consumidor. |

**Fontes:** `TRANSCRICAO.md` `[09:02]`–`[09:03]`; `[09:08]`; `[09:12]`–`[09:13]`; `[09:24]`–`[09:26]`; `[09:37]`–`[09:40]`; `[09:43]`–`[09:44]`.

## 6. Requisitos funcionais

### RF-01 — Notificar mudança de status

Quando o status de um pedido mudar, o OMS deve gerar um evento outbound `order.status_changed` para cada webhook ativo do cliente que esteja inscrito no novo status.

### RF-02 — Cadastrar webhook

Um usuário autenticado deve poder cadastrar um webhook por `POST`, informando a URL HTTPS, o cliente e a lista de status de interesse.

### RF-03 — Gerar secret na criação

O OMS deve gerar uma secret exclusiva para o endpoint e devolvê-la na criação da configuração.

### RF-04 — Editar webhook

Um usuário autenticado deve poder alterar a configuração por `PATCH`, incluindo URL, estado e filtros suportados pelo contrato.

### RF-05 — Remover webhook

Um usuário autenticado deve poder remover uma configuração por `DELETE`.

### RF-06 — Listar webhooks por cliente

Um usuário autenticado deve poder consultar por `GET` as configurações associadas a um cliente.

### RF-07 — Filtrar por status

Cada webhook deve aceitar uma lista dos status que deseja receber. Se nenhum webhook do cliente estiver inscrito no novo status, o OMS não deve registrar entrega para esse endpoint.

### RF-08 — Consultar histórico de entregas

O OMS deve disponibilizar `GET /webhooks/:id/deliveries` com o histórico de entregas do webhook, incluindo sucesso ou falha, payload, resposta e tempo de resposta.

### RF-09 — Rotacionar secret

O cliente deve poder solicitar uma nova secret pela API. A secret anterior deve permanecer válida em paralelo por 24 horas.

### RF-10 — Assinar notificações

Cada entrega deve incluir assinatura HMAC-SHA256 calculada sobre o corpo da requisição com a secret do endpoint.

### RF-11 — Identificar evento e endpoint

Cada notificação deve incluir `X-Event-Id` com UUID único e `X-Webhook-Id` com o identificador da configuração.

### RF-12 — Informar timestamp de envio

Cada notificação deve incluir `X-Timestamp`, permitindo que o consumidor aplique sua própria proteção contra replay.

### RF-13 — Retentar falhas e persistir DLQ

Entregas malsucedidas devem seguir a política de tentativas definida e, depois de esgotada, ser persistidas em uma DLQ separada com payload, motivo e timestamp.

### RF-14 — Reprocessar DLQ

Um usuário com role `ADMIN` deve poder solicitar replay manual por `POST /admin/webhooks/dead-letter/:id/replay`, recolocando o evento para processamento.

### RF-15 — Auditar replay

O OMS deve registrar qual usuário administrativo solicitou cada replay.

### RF-16 — Entregar snapshot do evento

O payload deve representar o estado existente no momento da mudança de status e conter `event_id`, `event_type`, timestamp ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e `total_cents`, sem itens.

### RF-17 — Permitir consulta aos detalhes do pedido

O evento deve manter o identificador necessário para o cliente consultar detalhes pela API existente `GET /orders/:id`.

**Fontes:** `TRANSCRICAO.md` `[09:18]`–`[09:25]`; `[09:31]`–`[09:36]`; `[09:42]`–`[09:45]`; `[09:51]`–`[09:52]`.

## 7. Requisitos não funcionais

| ID | Requisito |
| --- | --- |
| RNF-01 | A primeira tentativa deve ocorrer normalmente em menos de 10 segundos após a mudança de status. |
| RNF-02 | A entrega não deve bloquear nem causar rollback por indisponibilidade do consumidor. |
| RNF-03 | O evento deve ser registrado atomicamente com a mudança de status. |
| RNF-04 | A entrega deve usar semântica at-least-once; o consumidor deve deduplicar por `X-Event-Id`. |
| RNF-05 | O polling inicial deve ocorrer a cada 2 segundos, em processo separado da API. |
| RNF-06 | A primeira fase deve usar um único worker, preservando ordem por pedido enquanto essa condição for mantida. |
| RNF-07 | A política decidida é de 5 tentativas e intervalos de `1m/5m/30m/2h/12h`; a interpretação exata entre tentativa inicial e retries deve permanecer aberta até alinhamento. |
| RNF-08 | Cada chamada HTTP deve expirar após 10 segundos e então ser tratada como falha. |
| RNF-09 | Somente URLs HTTPS devem ser aceitas. |
| RNF-10 | O payload deve ter no máximo 64 KB; exceder o limite deve produzir erro, não truncamento. |
| RNF-11 | Secrets devem ser exclusivas por endpoint e rotacionáveis. |
| RNF-12 | O módulo deve reutilizar autenticação, autorização, validação, erros, logs e padrões modulares existentes. |
| RNF-13 | Erros de domínio do módulo devem usar o prefixo `WEBHOOK_*`. |
| RNF-14 | A implementação deve caber em três sprints, incluindo dois dias úteis para revisão de segurança antes do deploy. |

**Fontes:** `TRANSCRICAO.md` `[09:02]`; `[09:06]`–`[09:24]`; `[09:28]`–`[09:30]`; `[09:42]`; `[09:45]`–`[09:49]`.

## 8. Decisões e trade-offs principais

| Decisão | Benefício | Trade-off/limitação |
| --- | --- | --- |
| Outbox no MySQL existente | Atomicidade com a mudança do pedido e ausência de nova infraestrutura. | Exige worker por polling e gestão do crescimento das tabelas. |
| Worker separado com polling de 2 segundos | Isola a API e atende a meta de menos de 10 segundos. | Introduz atraso de polling e um processo adicional para operar. |
| Single worker na primeira fase | Simplifica processamento e preserva ordem por pedido no desenho inicial. | Limita escala; múltiplos workers e particionamento ficam adiados. |
| Retry finito e DLQ separada | Recupera indisponibilidades temporárias sem manter eventos eternamente pendentes. | Pode exigir replay manual quando a indisponibilidade superar a janela. |
| HMAC-SHA256 com secret por endpoint | Permite autenticar origem e integridade e limita o impacto do vazamento de uma secret. | Exige distribuição e rotação segura das secrets pelo cliente. |
| At-least-once com `X-Event-Id` | Evita a complexidade de exactly-once e permite recuperação de falhas. | Duplicatas são possíveis; deduplicação é responsabilidade do consumidor. |
| Snapshot enxuto na outbox | Preserva o estado histórico e limita o tamanho da entrega. | O consumidor precisa consultar a API para obter itens e outros detalhes. |

Detalhes arquiteturais e consequências estão registrados nos [ADRs](adrs/) e no [RFC](RFC.md).

## 9. Dependências

### 9.1 Dependências do produto e operação

- Clientes precisam expor endpoints HTTPS acessíveis pelo OMS.
- Consumidores precisam validar HMAC-SHA256 e deduplicar eventos por `X-Event-Id`.
- Clientes que precisarem de detalhes completos devem usar `GET /orders/:id`.
- A equipe deve documentar o contrato no portal de desenvolvedores.
- Sofia deve realizar revisão de segurança por pelo menos dois dias úteis antes do deploy.
- O worker separado precisa ser implantado e operado junto ao OMS.

### 9.2 Dependências do sistema existente

- Transação de mudança de status em `src/modules/orders/order.service.ts`.
- Máquina de estados em `src/modules/orders/order.status.ts`.
- Autenticação e autorização já usadas nas rotas.
- `PrismaClient` e MySQL existentes.
- Validação Zod, `AppError`, middleware centralizado de erros e logger Pino.

**Fontes:** `TRANSCRICAO.md` `[09:11]`; `[09:25]`–`[09:30]`; `[09:40]`–`[09:47]`; código inventariado em `docs/INVENTARIO-EVIDENCIAS.md`.

## 10. Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação prevista |
| --- | --- | --- | --- |
| Consumidor indisponível ou lento | Alta — indisponibilidades de até duas horas já ocorreram em clientes | Médio | Timeout de 10 segundos, retry com backoff, DLQ e replay administrativo. |
| Entrega duplicada | Média — inerente à garantia at-least-once | Médio | `X-Event-Id` único e orientação explícita para deduplicação pelo consumidor. |
| Vazamento da secret em sistemas do cliente | Média — incidente semelhante já ocorreu | Alto | Secret exclusiva por endpoint e rotação com grace period de 24 horas. |
| Evento ausente após mudança de status | Baixa após adoção da outbox | Alto | Inserção na outbox dentro da mesma transação da mudança do pedido. |
| Crescimento da outbox e degradação do polling | Média | Médio | Índices por status e `created_at`; política de arquivamento permanece como trabalho futuro. |
| Backlog superior à capacidade do single worker | Probabilidade não definida | Alto | Observar métricas; registrar escala horizontal/particionamento e rate limiting como decisões futuras. |
| Ordem de eventos ao escalar workers | Baixa na primeira fase; cresce com escala futura | Médio | Manter single worker inicialmente e tratar particionamento por `order_id` antes de escalar. |
| Uso indevido do replay da DLQ | Baixa | Alto | Restringir a `ADMIN` e auditar o usuário responsável. |

As classificações qualitativas usam apenas evidências da reunião quando disponíveis. Não há dados históricos suficientes para quantificar a probabilidade do backlog.

## 11. Critérios de aceitação

- [ ] Uma mudança de status elegível registra o snapshot do evento na mesma transação da alteração do pedido.
- [ ] Falha na inserção do evento causa rollback da mudança de status.
- [ ] Nenhuma chamada ao endpoint do cliente ocorre dentro da transação do pedido.
- [ ] A primeira tentativa ocorre normalmente em menos de 10 segundos.
- [ ] Um webhook recebe somente os status configurados.
- [ ] Usuário autenticado consegue criar, editar, remover e listar configurações por cliente.
- [ ] A criação gera e devolve uma secret exclusiva do endpoint.
- [ ] URL não HTTPS é rejeitada.
- [ ] Entrega inclui corpo JSON menor ou igual a 64 KB e os headers `Content-Type`, `X-Event-Id`, `X-Webhook-Id`, `X-Timestamp` e `X-Signature`.
- [ ] O consumidor consegue validar a assinatura HMAC-SHA256.
- [ ] A rotação disponibiliza nova secret e mantém a anterior válida por 24 horas.
- [ ] Falhas seguem a política definida e chegam à DLQ após o esgotamento.
- [ ] Histórico exibe sucesso/falha, payload, resposta e duração.
- [ ] Somente `ADMIN` consegue executar replay da DLQ.
- [ ] O replay registra o usuário responsável.
- [ ] Uma entrega repetida mantém o mesmo `X-Event-Id`, permitindo deduplicação.
- [ ] O evento contém snapshot da mudança e não inclui os itens do pedido.
- [ ] Sofia conclui a revisão de segurança antes do deploy.

## 12. Estratégia de testes e validação

### 12.1 Validação funcional

- Testar CRUD autenticado de webhooks e associação ao cliente informado.
- Testar filtros com webhooks interessados e não interessados em cada mudança de status.
- Verificar histórico de entregas com sucesso, falha, payload, resposta e duração.
- Testar rotação de secret durante e depois do grace period.
- Testar replay da DLQ com usuário `ADMIN`, usuário sem permissão e auditoria.

### 12.2 Validação da entrega

- Simular endpoint saudável, lento, indisponível e respostas de falha.
- Confirmar timeout de 10 segundos.
- Confirmar política de tentativas e persistência final na DLQ.
- Forçar repetição e verificar manutenção do `X-Event-Id`.
- Medir se a primeira tentativa fica abaixo de 10 segundos em operação normal.
- Confirmar a ordem por pedido no cenário de single worker.

### 12.3 Validação de consistência

- Confirmar commit conjunto da mudança de status e do evento.
- Forçar erro na inserção da outbox e verificar rollback integral.
- Confirmar que nenhum evento é criado quando não houver webhook interessado.
- Alterar o pedido depois da criação do evento e verificar que o payload mantém o snapshot original.

### 12.4 Validação de segurança

- Rejeitar URLs HTTP.
- Validar HMAC correto e detectar payload adulterado.
- Confirmar isolamento das secrets entre endpoints.
- Rejeitar payload acima de 64 KB sem truncamento.
- Realizar revisão de segurança de HMAC e geração/rotação de secrets antes do deploy.

### 12.5 Questões que precisam de decisão antes dos testes finais

- Interpretação exata de “5 tentativas” em relação à chamada inicial e aos cinco intervalos informados.
- Classificação de respostas HTTP retentáveis e permanentes.
- Formato exato de `X-Signature` e regra de bytes/canonicalização do corpo.
- Mecânica de validação das duas secrets durante o grace period.
- Volume esperado e SLO operacional pós-lançamento.

## 13. Rastreabilidade e documentos relacionados

- [Inventário de evidências](INVENTARIO-EVIDENCIAS.md)
- [RFC](RFC.md)
- [FDD](FDD.md)
- [ADRs](adrs/)
- `TRANSCRICAO.md`

Este PRD descreve o problema, os usuários, o escopo e os resultados esperados. Contratos HTTP, modelo de dados, fluxos internos e detalhes de implementação pertencem ao FDD; alternativas arquiteturais e questões abertas pertencem ao RFC; justificativas isoladas das decisões pertencem aos ADRs.
