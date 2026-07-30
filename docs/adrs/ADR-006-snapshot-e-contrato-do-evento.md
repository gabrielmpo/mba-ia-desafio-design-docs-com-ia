# ADR-006 — Snapshot e contrato do evento

- **Status:** Aceita
- **Decisores:** Larissa, Diego, Bruno e Marcos

## Contexto

Um pedido pode continuar mudando depois que um evento entra na outbox. Se o worker carregasse o pedido apenas no momento do envio, o conteúdo poderia representar um estado diferente daquele que originou a notificação.

## Decisão

Persistir na outbox o payload JSON já renderizado no momento da transação.

O evento `order.status_changed` conterá:

- `event_id`;
- `event_type`;
- `timestamp` ISO 8601;
- `order_id` e `order_number`;
- `from_status` e `to_status`;
- `customer_id`;
- `total_cents`.

Itens do pedido não serão incluídos. O cliente poderá consultar `GET /orders/:id` para obter detalhes. O payload terá limite de 64 KB; acima disso, a geração falhará e a transação será revertida.

## Alternativas consideradas

### Guardar apenas `order_id`

Descartada porque o conteúdo seria renderizado com o estado mais recente e poderia não refletir a transição original.

### Incluir todos os itens

Descartada para manter o evento pequeno, estável e focado na mudança de status.

## Consequências

### Positivas

- evento historicamente consistente;
- retries enviam exatamente o mesmo conteúdo;
- assinatura HMAC é estável;
- worker não precisa consultar e montar novamente o pedido.

### Negativas

- duplicação de alguns dados na outbox;
- evolução do schema exige versionamento compatível;
- erro de renderização impede a própria mudança de status, por participar da transação.

## Rastreabilidade

- `TRANSCRICAO.md`: `[09:23]`–`[09:24]`, `[09:43]`–`[09:45]`, `[09:51]`–`[09:52]`.
- Código relacionado: `src/modules/orders/order.service.ts`.

