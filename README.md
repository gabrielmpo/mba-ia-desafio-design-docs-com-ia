# Da reunião ao pacote de Design Docs

## Sobre o desafio

Este repositório apresenta a documentação da feature **Sistema de Webhooks de
Notificação de Pedidos** para um Order Management System (OMS). O trabalho
partiu de uma transcrição de reunião técnica e do código existente para
produzir um pacote de Design Docs acionável, sem implementar ou alterar a
aplicação.

O principal requisito de qualidade foi a rastreabilidade: requisitos,
decisões, restrições, alternativas e exclusões precisavam ter origem
identificável na [`TRANSCRICAO.md`](TRANSCRICAO.md) ou em um caminho real do
código. Informações plausíveis, mas não decididas, foram mantidas como
questões abertas ou propostas não normativas.

## Ferramentas de IA utilizadas

- **ChatGPT em Work Mode, com Codex**: exploração do repositório, classificação
  das evidências, elaboração e revisão crítica dos documentos, validações e
  manutenção do contexto entre as etapas.
- **Conector do GitHub no ChatGPT**: leitura do estado real do fork, criação de
  branches, commits e pull requests, além da conferência dos arquivos
  publicados.

Como apoio à validação, também foi usado **Markdownlint** para checar a
estrutura Markdown do FDD. A IA foi tratada como ferramenta de produção
assistida: cada resultado passou por confronto com as fontes antes de ser
aceito.

## Workflow adotado

O trabalho foi dividido em etapas pequenas e revisáveis:

1. leitura do enunciado, da transcrição e da estrutura do OMS;
2. criação de um inventário com 122 evidências classificadas;
3. produção de sete ADRs para registrar as decisões isoladamente;
4. elaboração do RFC no nível arquitetural, referenciando os ADRs;
5. elaboração e auditoria do FDD contra a transcrição e o código;
6. consolidação do PRD no nível de produto e negócio;
7. construção do Tracker com 99 itens rastreados;
8. documentação do processo neste README;
9. auditoria final item a item contra os critérios de aceite.

Cada grupo de documentos foi publicado em uma branch própria e submetido por
pull request. Essa separação permitiu revisar o escopo de cada mudança e
garantir que nenhum arquivo de aplicação fosse alterado.

### Organização da interação com a IA

Em vez de solicitar todos os documentos de uma vez, os prompts delimitaram o
papel de cada artefato e exigiram evidência para cada afirmação. O inventário
funcionou como base comum, enquanto o Tracker foi usado como verificação
transversal contra alucinações.

Quando uma decisão não aparecia explicitamente nas fontes, a instrução era não
preencher a lacuna por plausibilidade técnica. O item deveria ser removido,
marcado como questão aberta ou descrito como proposta não normativa.

## Prompts customizados

### 1. Inventário de evidências

```text
Analise integralmente TRANSCRICAO.md e o código do OMS antes de produzir
qualquer Design Doc. Crie um inventário de evidências classificando cada item
como contexto, requisito funcional, requisito não funcional, decisão,
alternativa descartada, fora de escopo, questão aberta, risco ou evidência de
código. Para a transcrição, registre timestamp e falante; para o código,
registre o caminho real do arquivo. Não transforme sugestão, hipótese ou item
adiado em requisito. Se não houver fonte identificável, não inclua o item como
fato.
```

### 2. Separação entre RFC e FDD

```text
Produza o RFC do sistema de webhooks em nível arquitetural, com TL;DR,
contexto, proposta geral, alternativas reais descartadas, questões abertas,
impactos, riscos e links para os ADRs. Mantenha o RFC conciso e não antecipe
detalhes de implementação. Fluxos passo a passo, contratos HTTP, modelo de
dados, matriz WEBHOOK_*, observabilidade e integração arquivo a arquivo
pertencem ao FDD. Toda afirmação normativa deve ser rastreável à transcrição
ou ao código.
```

### 3. Auditoria contra decisões inventadas

```text
Audite o FDD linha por linha contra TRANSCRICAO.md, o inventário, os ADRs, o
RFC e o código. Classifique cada detalhe técnico como: (1) confirmado por
timestamp ou caminho real; (2) proposta de implementação não normativa; ou
(3) não rastreável. Preserve (1), identifique explicitamente (2) e remova ou
transforme (3) em questão aberta. Procure especialmente números, defaults,
algoritmos de locking, classificação de erros HTTP, proteção de secrets,
formato de assinatura, paginação e regras de autorização.
```

### 4. Tracker quantitativo

```text
Monte docs/TRACKER.md usando exatamente as colunas ID, Documento, Tipo,
Conteúdo (resumo), Fonte e Localização. Consolide os identificadores do
inventário sem criar novos requisitos. Para Fonte = TRANSCRICAO, exija
localização no formato [hh:mm] Falante; para Fonte = CODIGO, exija um caminho
existente. Calcule a cobertura e a proporção de linhas da transcrição e sinalize
qualquer item documental sem origem verificável.
```

## Iterações e ajustes

Foram realizadas **10 iterações principais** de produção e revisão:

1. O pedido inicial por documentos foi precedido por um inventário de 122
   evidências para reduzir omissões e alucinações.
2. As decisões foram separadas em sete ADRs antes do RFC, evitando que o RFC
   virasse uma lista de decisões sem contexto.
3. O primeiro desenho do RFC estava detalhado demais; contratos e fluxos foram
   deslocados para o FDD.
4. O FDD foi cruzado com caminhos reais do OMS para substituir referências
   genéricas por integrações concretas.
5. A releitura do enunciado revelou que detalhes tecnicamente plausíveis
   estavam sendo apresentados como decisões sem fonte explícita.
6. A auditoria reclassificou lote de 20, `SKIP LOCKED`, AES-GCM, classificação
   de códigos HTTP, paginação e formato da assinatura como questões abertas ou
   propostas não normativas.
7. A autorização do CRUD foi corrigida de `ADMIN/OPERATOR` para qualquer
   usuário autenticado, mantendo o replay exclusivo de `ADMIN`.
8. A interpretação de “cinco tentativas” como seis chamadas totais foi
   removida, pois a reunião deixa ambígua a relação entre a chamada inicial e
   os cinco intervalos de backoff.
9. O PRD foi revisado para permanecer no nível de produto, sem duplicar
   contratos e decisões de implementação do FDD.
10. O Tracker normalizou 99 itens: 79 com timestamp e falante da transcrição e
    20 evidências distribuídas por 11 caminhos reais do código.

Esses ciclos transformaram conteúdos inicialmente plausíveis, porém
insuficientemente fundamentados, em documentos com fronteiras claras e
rastreabilidade verificável.

## Como navegar a entrega

Ordem sugerida de leitura:

1. [`docs/PRD.md`](docs/PRD.md) — problema, público, objetivos, escopo e
   requisitos;
2. [`docs/RFC.md`](docs/RFC.md) — proposta arquitetural, alternativas e
   questões abertas;
3. [`docs/adrs/`](docs/adrs/) — sete decisões arquiteturais isoladas;
4. [`docs/FDD.md`](docs/FDD.md) — especificação detalhada de implementação;
5. [`docs/TRACKER.md`](docs/TRACKER.md) — origem de cada item na transcrição ou
   no código;
6. [`docs/INVENTARIO-EVIDENCIAS.md`](docs/INVENTARIO-EVIDENCIAS.md) —
   levantamento completo usado como base de produção.

### Artefatos entregues

| Artefato | Finalidade |
| --- | --- |
| [`docs/PRD.md`](docs/PRD.md) | Requisitos de produto e critérios de sucesso |
| [`docs/RFC.md`](docs/RFC.md) | Proposta técnica para revisão |
| [`docs/FDD.md`](docs/FDD.md) | Instruções detalhadas de implementação |
| [`docs/adrs/`](docs/adrs/) | Registro de sete decisões arquiteturais |
| [`docs/TRACKER.md`](docs/TRACKER.md) | Matriz transversal de rastreabilidade |
| [`docs/INVENTARIO-EVIDENCIAS.md`](docs/INVENTARIO-EVIDENCIAS.md) | Catálogo das evidências extraídas |

O repositório-base e o enunciado original do desafio estão disponíveis em
[`devfullcycle/mba-ia-desafio-design-docs-com-ia`](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).
