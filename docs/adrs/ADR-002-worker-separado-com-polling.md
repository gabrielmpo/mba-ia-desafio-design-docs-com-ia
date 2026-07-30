# ADR-002 — Worker separado com polling

- **Status:** Aceita
- **Decisores:** Larissa, Bruno e Diego

## Contexto

Os eventos da outbox precisam ser processados sem bloquear a API. A expectativa dos clientes considera tempo real qualquer entrega abaixo de dez segundos. O MySQL não oferece mecanismo equivalente ao `LISTEN/NOTIFY` do PostgreSQL para despertar um consumidor externo.

O processo HTTP atual inicia em `src/server.ts`, conecta-se pelo Prisma configurado em `src/config/database.ts` e possui ciclo próprio de encerramento.

## Decisão

Executar um worker Node.js em processo separado da API, com nova entry point `src/worker.ts`, reutilizando a mesma `DATABASE_URL`, mas com um `PrismaClient` próprio do processo.

O worker fará polling a cada dois segundos e buscará pequenos lotes de eventos pendentes ordenados por `created_at`. Nesta fase haverá uma única instância, preservando a ordem por pedido enquanto o processamento for serial.

## Alternativas consideradas

### Worker dentro do processo da API

Descartado porque acopla o ciclo de vida do consumidor ao servidor HTTP e dificulta escalar, reiniciar e observar cada carga separadamente.

### Trigger de banco

Descartada porque a trigger executa SQL, mas não notifica de forma adequada um processo externo.

### Broker ou Redis

Descartado nesta fase pelo custo operacional adicional.

## Consequências

### Positivas

- isolamento entre tráfego HTTP e entregas;
- implantação, reinício e observabilidade independentes;
- latência de polling compatível com o requisito de menos de dez segundos;
- reutilização da stack Node.js, Prisma, Pino e configuração existente.

### Negativas

- até dois segundos de espera antes do processamento;
- mais um processo para implantar e monitorar;
- single worker limita throughput;
- múltiplos workers futuros exigirão locking ou particionamento por `order_id`.

## Rastreabilidade

- `TRANSCRICAO.md`: `[09:08]`–`[09:13]`.
- Código: `src/server.ts`, `src/config/database.ts`, `src/shared/logger/index.ts`.

