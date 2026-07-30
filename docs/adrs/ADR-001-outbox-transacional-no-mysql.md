# ADR-001 — Outbox transacional no MySQL

- **Status:** Aceita
- **Data:** reunião técnica registrada em `TRANSCRICAO.md`
- **Decisores:** Larissa, Bruno e Diego

## Contexto

A mudança de status de um pedido já ocorre dentro de uma transação Prisma que atualiza `Order`, registra `OrderStatusHistory` e, conforme a transição, debita ou repõe estoque. O ponto de integração está em `src/modules/orders/order.service.ts`, no método `OrderService.changeStatus`.

Disparar o webhook de forma síncrona dentro dessa transação faria a operação depender da latência e disponibilidade do endpoint do cliente. Registrar o evento fora da transação, por outro lado, permitiria que o status fosse confirmado sem que a notificação correspondente existisse.

## Decisão

Adotar o padrão Transactional Outbox usando o MySQL já existente.

Ao alterar o status, `changeStatus` chamará uma função do módulo de webhooks que recebe o `Prisma.TransactionClient`. Na mesma transação serão persistidos:

1. o novo status do pedido;
2. o histórico da transição;
3. os ajustes de estoque aplicáveis;
4. um evento por endpoint interessado na tabela `webhook_outbox`.

Se a criação da outbox falhar, toda a mudança será revertida. O evento terá UUID e payload em snapshot, renderizado no momento da inserção.

## Alternativas consideradas

### Chamada HTTP síncrona

Descartada porque aumenta o tempo da transação e faz uma indisponibilidade externa bloquear ou reverter mudanças válidas de pedido.

### Redis Streams ou broker dedicado

Descartado nesta fase por exigir nova infraestrutura e operação para um volume ainda desconhecido. O MySQL existente atende ao requisito de entrega em menos de dez segundos.

### Persistência do evento depois do commit

Descartada porque cria uma janela em que a mudança de status existe sem evento correspondente.

## Consequências

### Positivas

- atomicidade entre mudança de estado e geração do evento;
- nenhuma dependência externa dentro da transação;
- reaproveitamento do MySQL e do Prisma;
- payload imutável e coerente com o estado que originou o evento.

### Negativas

- crescimento da tabela de outbox, que exigirá índices e política futura de arquivamento;
- maior duração da transação devido às inserções adicionais;
- necessidade de worker e mecanismos próprios de concorrência e recuperação.

## Rastreabilidade

- `TRANSCRICAO.md`: `[09:03]`–`[09:10]`, `[09:40]`–`[09:41]`, `[09:51]`–`[09:52]`.
- Código: `src/modules/orders/order.service.ts`.

