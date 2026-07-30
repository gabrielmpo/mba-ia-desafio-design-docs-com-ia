# ADR-004 — Autenticação HMAC e rotação de secret

- **Status:** Aceita
- **Decisores:** Sofia, Larissa, Bruno e Diego

## Contexto

Os webhooks transportam dados de pedidos para sistemas externos. O destinatário precisa validar a origem e a integridade do corpo, sem depender de credenciais globais que ampliem o impacto de um vazamento.

## Decisão

Assinar o corpo JSON exato de cada requisição com HMAC-SHA256 e enviar a assinatura no header `X-Signature`.

Cada endpoint terá secret exclusiva, gerada pelo sistema na criação. A rotação produzirá uma nova secret e manterá a anterior válida por 24 horas. Após o grace period, somente a nova será aceita. A URL cadastrada deverá usar HTTPS.

Também serão enviados `X-Event-Id`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`.

## Alternativas consideradas

### Secret global

Descartada porque um único vazamento comprometeria todos os clientes.

### Assinatura assimétrica

Não adotada nesta fase: simplificaria a distribuição de verificação, mas acrescentaria gestão de chaves e complexidade incompatíveis com o escopo atual.

### Envio sem assinatura, confiando apenas em TLS

Descartado porque TLS protege o transporte, mas não oferece ao cliente prova aplicacional da origem do payload.

## Consequências

### Positivas

- autenticação e integridade com bibliotecas amplamente disponíveis;
- isolamento de credenciais por endpoint;
- rotação sem interrupção imediata da integração;
- suporte a mitigação de replay pelo timestamp.

### Negativas

- armazenamento e exibição de secrets exigem cuidado;
- cliente precisa implementar verificação sobre os mesmos bytes;
- durante 24 horas duas secrets permanecem válidas;
- o contrato deve especificar canonicalização e formato da assinatura.

## Rastreabilidade

- `TRANSCRICAO.md`: `[09:19]`–`[09:24]`, `[09:44]`–`[09:45]`.

