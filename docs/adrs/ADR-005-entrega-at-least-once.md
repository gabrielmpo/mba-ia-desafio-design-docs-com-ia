# ADR-005 — Entrega at-least-once

- **Status:** Aceita
- **Decisores:** Larissa, Diego, Sofia e Marcos

## Contexto

Depois de enviar uma requisição, o worker pode perder a resposta ou falhar antes de registrar o sucesso. Não há coordenação transacional entre o banco do OMS e o sistema do cliente; portanto, eliminar duplicatas do lado do emissor não é garantível em todos os cenários.

## Decisão

Oferecer garantia **at-least-once**. Cada evento receberá um UUID imutável no momento da inserção na outbox. O identificador será enviado no corpo como `event_id` e no header `X-Event-Id`.

Os consumidores deverão tornar seu processamento idempotente e deduplicar usando esse identificador. Uma nova tentativa do mesmo evento preservará o mesmo `event_id`.

## Alternativas consideradas

### Exactly-once

Descartada porque exigiria protocolo de coordenação com cada consumidor e ainda não eliminaria todos os efeitos duplicados em sistemas externos.

### At-most-once

Descartada porque evitar retry após falhas ambíguas poderia perder notificações.

## Consequências

### Positivas

- evita perda silenciosa em falhas transitórias ou ambíguas;
- contrato simples e compatível com práticas comuns de webhooks;
- deduplicação determinística pelo consumidor.

### Negativas

- clientes podem receber o mesmo evento mais de uma vez;
- idempotência passa a ser responsabilidade explícita do consumidor;
- documentação e histórico de entregas precisam distinguir tentativas de eventos.

## Rastreabilidade

- `TRANSCRICAO.md`: `[09:24]`–`[09:26]`, `[09:51]`.

