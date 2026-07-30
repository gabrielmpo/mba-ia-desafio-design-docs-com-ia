# ADR-007 — Reuso dos padrões do OMS

- **Status:** Aceita
- **Decisores:** Larissa, Bruno e Diego

## Contexto

O OMS organiza domínios em módulos com controller, service, repository, routes e schemas. Já possui autenticação, validação Zod, erros de domínio, middleware de erro centralizado, Prisma e logging Pino. Introduzir convenções paralelas aumentaria o custo de manutenção.

## Decisão

Implementar webhooks como `src/modules/webhooks`, seguindo a estrutura dos módulos existentes:

- controller para adaptação HTTP;
- service para casos de uso;
- repository para persistência;
- routes para composição Express;
- schemas Zod para entrada;
- processor/worker para entrega.

Erros do domínio reutilizarão `AppError` e o middleware central, com códigos prefixados por `WEBHOOK_`. Logging usará o Pino existente. Rotas normais usarão autenticação existente; replay de DLQ reutilizará `requireRole(ADMIN)`.

O worker terá `PrismaClient` próprio por ser outro processo, mas compartilhará configuração, modelos e logger.

## Alternativas consideradas

### Serviço isolado em outro repositório

Descartado porque fragmentaria a feature, duplicaria padrões e infraestrutura e dificultaria a transação com a outbox no mesmo banco.

### Arquitetura interna específica para webhooks

Descartada porque não há benefício que compense uma segunda convenção de módulos, erros e logs.

## Consequências

### Positivas

- menor curva de aprendizado e revisão;
- integração natural com middlewares e composição existentes;
- códigos de erro e logs consistentes;
- menor quantidade de dependências novas.

### Negativas

- o módulo herda limitações e acoplamentos atuais do monólito;
- o worker compartilha tipos e configuração com a API;
- extração futura para serviço independente exigirá redefinir fronteiras.

## Rastreabilidade

- `TRANSCRICAO.md`: `[09:27]`–`[09:30]`, `[09:35]`–`[09:36]`.
- Código: `src/modules/orders/order.controller.ts`, `src/modules/orders/order.routes.ts`, `src/modules/orders/order.service.ts`, `src/server.ts`, `src/shared/errors/index.ts`, `src/shared/logger/index.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/validate.middleware.ts`.

