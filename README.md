# Da Reunião ao Código: O Processo de Produção de Design Docs com Inteligência Artificial

Este repositório contém a documentação técnica de arquitetura e produto para a implementação de um **Sistema de Webhooks de Notificação de Pedidos** integrado ao Order Management System (OMS) existente. O pacote de Design Docs foi produzido de forma sistemática a partir da análise detalhada do código da aplicação e da transcrição literal de uma reunião de alinhamento técnico (`TRANSCRICAO.md`).

---

## 1. Sobre o Desafio

O desafio consiste em assumir o papel de **Maestro de Inteligência Artificial** para preencher a lacuna entre uma discussão técnica em tempo real (registrada em áudio/transcrição) e a especificação técnica formal que guiará a implementação do time de engenharia. O principal objetivo foi projetar um sistema de webhooks assíncrono, transacional, resiliente e altamente seguro sobre uma base de código OMS pré-existente escrita em Node.js, TypeScript, Express e Prisma/MySQL, que originalmente não possuía nenhum mecanismo de envio de eventos ou mensageria.

A produção exigiu o mapeamento rigoroso de requisitos de produto (no PRD), propostas arquiteturais abstratas e alternativas rejeitadas (no RFC), especificações granulares de implementação e contratos de API (no FDD), decisões técnicas estruturais isoladas (nos ADRs) e a amarração absoluta de cada detalhe em uma matriz de rastreabilidade transversal (no Tracker). Esse processo garante que nenhuma decisão de engenharia tenha sido inventada ("alucinada") e que todo requisito atenda fielmente às discussões e restrições reais de custos, infraestrutura e prazos mencionadas na transcrição.

---

## 2. Ferramentas de IA Utilizadas

A execução deste projeto foi realizada exclusivamente utilizando a família de modelos **Gemini** (como ferramenta única de Inteligência Artificial):

* **Gemini 1.5 Pro / Flash (via Gemini CLI & Google AI Studio)**: Atuou de forma integral em todo o ciclo de vida da produção da entrega. O modelo foi responsável pela leitura e processamento analítico da transcrição de 55 minutos (`TRANSCRICAO.md`) e da base de código do OMS escrita em TypeScript e Prisma, bem como pela estruturação e redação final dos artefatos técnicos (PRD, RFC, FDD, ADRs, Tracker e este README). Também serviu como ferramenta de validação sistemática, cruzando recursivamente os textos gerados com a checklist de critérios de aceitação para assegurar a consistência técnica total e ausência de alucinações.

---

## 3. Workflow Adotado

Para evitar que a documentação ficasse superficial ou redundante entre os arquivos, adotamos uma abordagem de **baixo para cima (Bottom-Up)**, projetando primeiro os blocos estruturais de código e subindo sequencialmente para as definições de produto:

```
[Mapeamento Inicial] ──> [ADRs (Decisões)] ──> [RFC (Consenso)] ──> [FDD (Contratos/Código)] ──> [PRD (Negócio)] ──> [Tracker & README]
```

1. **Mapeamento e Extração Inicial**: Alimentamos o Gemini com a base de código do OMS e a `TRANSCRICAO.md` para extrair um inventário bruto de preocupações técnicas (ex: preocupação de Sofia com HMAC, ressalva de Bruno sobre custos de broker e a exigência de Marcos sobre latência).
2. **Definição dos ADRs (Architecture Decision Records)**: Em vez de começar pelo PRD, geramos primeiro os 6 ADRs estruturais na pasta `docs/adrs/`. Definir o padrão Transactional Outbox (ADR-001), a política de retry/DLQ (ADR-002), a criptografia HMAC (ADR-003), a idempotência via `X-Event-Id` (ADR-004), o Polling Worker (ADR-005) e o reuso de padrões (ADR-006) formou o alicerce sólido de como o sistema funcionaria na prática.
3. **Redação do RFC (Request for Comments)**: Com as decisões tomadas, escrevemos o RFC sob a ótica de uma proposta técnica aberta para a equipe (composta por Diego, Sofia, Bruno, Larissa e Marcos). Concentramos este documento no "porquê" da arquitetura, descrevendo de forma concisa as abordagens propostas, as alternativas descartadas na reunião (mensageria externa com RabbitMQ/Redis e envio síncrono na API) e os pontos deixados em aberto (como Rate Limiting e concorrência).
4. **Detalhando a Engenharia no FDD (Feature Design Document)**: Com a arquitetura acordada, elaboramos a especificação de baixo nível. Mapeamos os contratos públicos de 4 endpoints HTTP, desenhamos diagramas de fluxo de dados, criamos uma matriz estrita de códigos de erros (`WEBHOOK_`) e detalhamos de forma exata a **Integração com o Sistema Existente**, citando as linhas e arquivos que seriam estendidos (como `order.service.ts` e `schema.prisma`).
5. **Consolidação do PRD (Product Requirement Document)**: Por fim, formalizamos a visão de produto. Definimos métricas de sucesso com metas numéricas agressivas (ex: latência de entrega p95 < 10s e redução de 80% em polling ineficiente), organizamos os requisitos funcionais e não funcionais mapeados, limitamos o escopo exclusivo (ex: descartando dashboards visuais e notificações de email) e avaliamos riscos práticos.
6. **Mapeamento de Rastreabilidade (Tracker)**: Varremos cada um dos documentos de design e registramos 35 itens chave em uma matriz estruturada. Mapeamos cada requisito e decisão ao seu timestamp cronológico exato do falante na transcrição (atendendo à exigência de no mínimo 70% de fontes na transcrição) ou ao caminho físico do arquivo existente no código.

---

## 4. Prompts Customizados

Abaixo estão descritos os dois principais prompts de engenharia de contexto utilizados nas iterações com a IA para extrair valor real e evitar que os documentos gerassem estruturas vazias ou fora de conformidade.

### Prompt 1: Engenharia Reversa de Decisões Técnicas (ADR/RFC Focus)
Este prompt foi desenvolvido para impedir que a IA inventasse soluções modernas de mercado (como Kafka ou AWS Lambda) que contrariassem a transcrição técnica ou que gerassem decisões genéricas demais.

```markdown
Você é o Tech Lead da equipe do OMS. Analise com extremo rigor o arquivo `TRANSCRICAO.md` e a estrutura de pastas do projeto (onde usamos Prisma, MySQL, Node e Express).
Sua missão é extrair exatamente as decisões técnicas tomadas na reunião, com seus respectivos motivadores e as restrições explícitas de custos, infraestrutura ou tempo levantadas pelo time.

Para cada decisão identificada (Outbox, Retries/DLQ, HMAC-SHA256, Polling Worker):
1. Quem propôs o item e qual foi o timestamp cronológico associado?
2. Quais alternativas foram rejeitadas e quais trade-offs foram expostos para justificar a rejeição (ex: restrições de broker levantadas pela Larissa/Diego)?
3. Como essa decisão se amarra especificamente com arquivos de código existentes no projeto?

Gere um rascunho de ADR estruturado no formato MADR para cada um dos pontos acima. Lembre-se: se uma decisão não tiver raiz direta em uma fala da transcrição ou no código atual, ela NÃO DEVE ser incluída. Seja focado em economia de recursos e pragmatismo técnico.
```

### Prompt 2: Alinhamento Granular de Integração de Código (FDD Focus)
Este prompt foi projetado para forçar a IA a analisar o código-fonte existente antes de desenhar os contratos de integração do FDD, garantindo o reuso perfeito das classes de tratamento de erro e middlewares de autenticação existentes.

```markdown
Você é o Engenheiro de Software Principal do projeto OMS. Leia com atenção os arquivos do repositório:
- `src/shared/errors/app-error.ts` (como as classes de exceção de negócio herdam dela e qual o formato do JSON de erro retornado)
- `src/middlewares/auth.middleware.ts` (como funciona o controle de roles e JWT)
- `src/modules/orders/order.service.ts` (especificamente o método `changeStatus` e como ele abre transações SQL usando o Prisma)
- `prisma/schema.prisma` (a modelagem de dados atual e como as tabelas relacionam-se usando UUID)

Escreva a seção "Integração com o Sistema Existente" para o Feature Design Document (FDD). Você deve citar exatamente pelo menos 4 arquivos físicos reais da árvore do repositório, descrevendo detalhadamente:
1. Como o `webhook_outbox` será populado dentro do bloco `this.prisma.$transaction` existente em `order.service.ts`, mantendo a atomicidade.
2. Como as rotas administrativas criadas para visualização de logs e replay da DLQ usarão o middleware `requireRole` e o JWT existente.
3. Como nossa nova matriz de erros de webhook (com o prefixo `WEBHOOK_`) estenderá o padrão `AppError` e como o manipulador de erros centralizado do Express (`error.middleware.ts`) responderá de forma padronizada.
4. Como estenderemos o banco usando migrations reais do Prisma sem impactar os modelos existentes.

Não invente pacotes que não existem no `package.json` (como o uso de 'bullmq' ou 'redis'). Toda a lógica deve rodar nativamente com as ferramentas existentes.
```

---

## 5. Iterações e Ajustes (Refinamento Crítico)

Durante as sessões de trabalho, a IA frequentemente gerou propostas inadequadas ou generalizadas. O refinamento humano ("papel do maestro") foi determinante para ajustar o design técnico em pelo menos duas iterações complexas:

### Ajuste 1: O "Tiro no Pé" das Chamadas HTTP na Transação do Banco
* **O Problema (Geração da IA)**: Na primeira rodada de geração do fluxo do FDD, a IA sugeriu que o método `OrderService.changeStatus` fizesse a inserção da intenção na outbox e, no mesmo bloco síncrono da transação do banco, tentasse disparar uma requisição HTTP HTTP-POST rápida para o parceiro. Se falhasse, a IA sugeriu que a transação continuasse ativa e o erro fosse ignorado.
* **A Correção Crítica**: Isso viola gravemente a confiabilidade de uma transação SQL e introduz risco severo de lentidão e timeout no banco caso o endpoint do cliente demore a responder. Instruímos a IA a revisar o fluxo de acordo com as falas de Diego em `[09:06]` e `[09:09]`: o fluxo HTTP de API do OMS apenas grava atomicamente o payload na tabela `webhook_outbox` como `PENDING` dentro da transação e encerra o fluxo imediatamente, devolvendo sucesso. Quem efetivamente lê e dispara a chamada de rede HTTP de forma 100% isolada e assíncrona é um Worker executando em processo separado (`npm run worker`) rodando em loop contínuo de polling a cada 2 segundos.

### Ajuste 2: Modelagem Temporal para Rotação de Secrets com Grace Period de 24h
* **O Problema (Geração da IA)**: Ao modelar o cadastro de webhooks e a rotação de segredos criptográficos no FDD e no ADR-003, a IA desenhou a rotação de chaves como um simples overwrite: ao acionar a rotação, o campo `secret` na tabela do banco de dados era imediatamente substituído pelo novo hash gerado.
* **A Correção Crítica**: Durante a reunião técnica, Sofia solicitou explicitamente em `[09:21]` que a rotação de segredos incluísse uma carência de segurança (grace period) de 24 horas para dar tempo ao parceiro comercial de atualizar seus ambientes de forma assíncrona sem interromper o serviço. Corrigimos a IA solicitando que ela modelasse a tabela do Prisma contendo tanto o campo `currentSecret` quanto o campo `previousSecret`, além de uma coluna `rotatedAt` contendo o timestamp do momento da rotação. Dessa forma, a validação de assinatura HMAC no middleware do cliente de destino (ou na nossa rota de simulação) conseguiria validar transições testando as duas chaves válidas dentro da janela de 24 horas.

---

## 6. Como Navegar a Entrega

Para revisar a especificação técnica de ponta a ponta, sugerimos seguir a ordem de leitura recomendada abaixo, estruturada do macro ao micro:

1. **Decisões Estruturais (Os ADRs)**: Comece navegando por `docs/adrs/`. Eles descrevem os fundamentos técnicos adotados.
   * [ADR-001: Padrão Outbox no MySQL](./docs/adrs/ADR-001-padrao-outbox-no-mysql.md) — Atomicidade de banco de dados e escrita do evento.
   * [ADR-002: Política de Retry com Backoff e DLQ](./docs/adrs/ADR-002-politica-de-retry-com-backoff-e-dlq.md) — Tratamento de erros de rede e persistência resiliente.
   * [ADR-003: Autenticação HMAC-SHA256 com Secret por Endpoint](./docs/adrs/ADR-003-autenticacao-hmac-sha256-com-secret-por-endpoint.md) — Protocolo de segurança e grace period de secrets.
   * [ADR-004: Garantia At-Least-Once com X-Event-Id](./docs/adrs/ADR-004-garantia-at-least-once-com-x-event-id.md) — Protocolo de idempotência no receptor.
   * [ADR-005: Worker em Processo Separado em Polling](./docs/adrs/ADR-005-worker-em-processo-separado-em-polling.md) — Desacoplamento assíncrono e loop de execução.
   * [ADR-006: Reuso de Padrões Existentes do Projeto](./docs/adrs/ADR-006-reuso-de-padroes-existentes-do-projeto.md) — Alinhamento técnico com a codebase do OMS.
2. **Proposta Arquitetural (O RFC)**: Leia `docs/RFC.md` para compreender o contexto do ecossistema, os metadados dos participantes envolvidos e os trade-offs das abordagens que foram descartadas pelo comitê técnico.
3. **Especificação de Baixo Nível (O FDD)**: Acesse `docs/FDD.md` para analisar as rotas de API, payloads de entrada/saída detalhados, códigos estritos de erro baseados em `WEBHOOK_`, tratamento de resiliência e a seção de **Integração com o Sistema Existente**.
4. **Alinhamento de Negócio (O PRD)**: Leia `docs/PRD.md` para verificar as regras de escopo inclusivo/exclusivo, métricas de sucesso com metas numéricas (p95, taxas de conversão) e plano de mitigação de riscos comerciais.
5. **Rastreabilidade (O Tracker)**: Utilize o [Tracker de Rastreabilidade](./docs/TRACKER.md) como uma tabela de referência cruzada para certificar-se de que cada requisito, contrato ou decisão técnica aponta com fidelidade absoluta a um timestamp de falante em `TRANSCRICAO.md` ou a um caminho de arquivo de código no repositório.

---

## 7. Referências e Enunciado Original

Caso deseje consultar as instruções originais do desafio, critérios de aceite estritos ou o cenário completo proposto pelo professor para o desenvolvimento deste projeto, acesse a [Matriz de Verificação e Checklist de Critérios de Aceite](./docs/REVISAO_REQUISITOS_CHECKLIST.md), que descreve individualmente cada meta mapeada e validada neste repositório.
