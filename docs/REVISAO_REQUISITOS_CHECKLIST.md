# Revisão de Requisitos e Checklist de Design Docs

Este documento serve como repositório de descarregamento (offload) de requisitos e validação sistemática de cada um dos artefatos produzidos no projeto: **PRD**, **RFC**, **FDD**, **ADRs** e **Tracker**. A verificação é realizada cruzando os requisitos oficiais do desafio (`README.md`), o conteúdo da reunião técnica (`TRANSCRICAO.md`) e a base de código do OMS (`src/` e `prisma/`).

---

## 1. Mapeamento de Requisitos (Offload do README.md)

Abaixo estão listados os critérios de aceitação exigidos pelo desafio para cada um dos documentos, servindo como nossa matriz de aprovação física.

### 1.1. Checklist do PRD (`docs/PRD.md`)
- [x] O arquivo existe na pasta correta e está formatado em Markdown.
- [x] Contém a seção: *Resumo e contexto da feature*.
- [x] Contém a seção: *Problema e motivação*.
- [x] Contém a seção: *Público-alvo e cenários de uso*.
- [x] Contém a seção: *Objetivos e métricas de sucesso* (incluindo pelo menos 1 métrica e meta quantitativa).
- [x] Contém a seção: *Escopo (inclusivo e exclusivo)*.
- [x] Contém a seção: *Requisitos funcionais* (mínimo de 8 identificados e detalhados).
- [x] Contém a seção: *Requisitos não funcionais*.
- [x] Contém a seção: *Decisões e trade-offs principais*.
- [x] Contém a seção: *Dependências*.
- [x] Contém a seção: *Riscos e mitigação* (mínimo de 2 riscos com probabilidade, impacto e mitigação).
- [x] Contém a seção: *Critérios de aceitação*.
- [x] Contém a seção: *Estratégia de testes e validação*.
- [x] A seção "Fora de escopo" lista pelo menos 2 itens explicitamente descartados ou adiados na reunião.

### 1.2. Checklist do RFC (`docs/RFC.md`)
- [x] O arquivo existe na pasta correta e está formatado em Markdown.
- [x] Contém a seção: *Metadados* (autor, status, data, revisores) mapeando os participantes reais da reunião.
- [x] Contém a seção: *Resumo executivo (TL;DR)*.
- [x] Contém a seção: *Contexto e problema*.
- [x] Contém a seção: *Proposta técnica* (visão arquitetural abstrata de alto nível).
- [x] Contém a seção: *Alternativas consideradas* (mínimo de 2 alternativas reais descartadas com seus respectivos trade-offs de descarte).
- [x] Contém a seção: *Questões em aberto* (mínimo de 2 pontos adiados ou "a observar").
- [x] Contém a seção: *Impacto e riscos*.
- [x] Contém a seção: *Decisões relacionadas* (contendo links funcionais para os ADRs correspondentes).
- [x] Referencia por meio de links relativos no mínimo 2 ADRs da pasta `docs/adrs/`.
- [x] O documento é conciso (2 a 4 páginas) e não duplica o detalhamento granular de implementação presente no FDD.

### 1.3. Checklist do FDD (`docs/FDD.md`)
- [x] O arquivo existe na pasta correta e está formatado em Markdown.
- [x] Contém a seção: *Contexto e motivação técnica*.
- [x] Contém a seção: *Objetivos técnicos*.
- [x] Contém a seção: *Escopo e exclusões*.
- [x] Contém a seção: *Fluxos detalhados* (criação do evento na outbox, processamento pelo worker, retry e DLQ).
- [x] Contém a seção: *Contratos públicos* (endpoints HTTP com payload de exemplo de request, response e status codes).
- [x] Contém a seção: *Matriz de erros previstos* (usando códigos consistentes com o prefixo `WEBHOOK_`).
- [x] Contém a seção: *Estratégias de resiliência* (timeouts, retries, backoff, fallback/DLQ).
- [x] Contém a seção: *Observabilidade* (citação de métricas, logs com Pino e tracing/correlation ID).
- [x] Contém a seção: *Dependências e compatibilidade* (ex: Prisma, MySQL, Node.js).
- [x] Contém a seção: *Critérios de aceite técnicos*.
- [x] Contém a seção: *Riscos e mitigação*.
- [x] Contém a seção obrigatória adicional: *Integração com o sistema existente*.
- [x] A seção de contratos públicos possui no mínimo 4 endpoints descritos em profundidade com request, response e status HTTP.
- [x] A seção "Integração com o sistema existente" nomeia pelo menos 4 caminhos de arquivo reais do código base e detalha a extensão ou reuso de cada um.

### 1.4. Checklist dos ADRs (`docs/adrs/ADR-NNN-*.md`)
- [x] A pasta `docs/adrs/` possui entre 5 e 8 arquivos com a nomenclatura correta (`ADR-NNN-titulo-em-kebab-case.md`).
- [x] Cada ADR possui as seções obrigatórias: *Status*, *Contexto*, *Decisão*, *Alternativas Consideradas* (mínimo 1) e *Consequências* (com trade-offs).
- [x] Pelo menos 1 ADR referencia explicitamente arquivos, módulos, classes ou padrões da codebase atual.
- [x] O conjunto cobre no mínimo 5 das 6 decisões principais:
  - [x] Decisão 1: Padrão Outbox no MySQL (`docs/adrs/ADR-001-padrao-outbox-no-mysql.md`)
  - [x] Decisão 2: Política de retry com backoff e DLQ (`docs/adrs/ADR-002-politica-de-retry-com-backoff-e-dlq.md`)
  - [x] Decisão 3: Autenticação HMAC-SHA256 com secret por endpoint (`docs/adrs/ADR-003-autenticacao-hmac-sha256-com-secret-por-endpoint.md`)
  - [x] Decisão 4: Garantia at-least-once com `X-Event-Id` (`docs/adrs/ADR-004-garantia-at-least-once-com-x-event-id.md`)
  - [x] Decisão 5: Worker em processo separado em polling (`docs/adrs/ADR-005-worker-em-processo-separado-em-polling.md`)
  - [x] Decisão 6: Reuso dos padrões existentes do projeto (`docs/adrs/ADR-006-reuso-de-padroes-existentes-do-projeto.md`)

### 1.5. Checklist do Tracker (`docs/TRACKER.md`)
- [x] O arquivo existe na pasta correta e está formatado no formato de tabela Markdown estipulado no `README.md`.
- [x] Pelo menos 80% dos itens identificáveis dos documentos têm linha correspondente no Tracker (35 itens cobertos).
- [x] Pelo menos 70% das linhas têm Fonte = `TRANSCRICAO` com timestamp válido e nome do falante (ex: `[09:17] Diego`) (coberto: 82.8%).
- [x] Pelo menos 5 linhas têm Fonte = `CODIGO` com caminho físico de arquivo real existente (6 linhas cobertas).

---

## 2. Validação Cruzada: Transcrição vs. Design Docs

Análise detalhada de consistência para garantir que nenhuma decisão registrada contradiz o diálogo ou introduz alucinações (requisitos inventados).

### 2.1. Confiabilidade e Atomicidade (Outbox)
* **Reunião**: Diego explica o padrão Outbox em `[09:06]`: a intenção do envio é gravada na tabela `webhook_outbox` como parte da mesma transação SQL que atualiza a ordem de compra.
* **Código**: `src/modules/orders/order.service.ts` contém o método transacional `changeStatus` que abre o `this.prisma.$transaction(async (tx) => { ... })`.
* **Docs**: PRD (PRD-FR-06, PRD-RNF-01), FDD (Seção 4.1, Seção 9 item 2) e ADR-001 alinham-se perfeitamente. Detalham que a inserção na outbox se dará após o `tx.orderStatusHistory.create` passando o client de transação `tx`.

### 2.2. Frequência de Polling e Latência
* **Reunião**: Diego sugere polling a cada 2 segundos em `[09:09]`, o que atende à janela de latência de menos de 10 segundos requisitada por Marcos em `[09:02]`.
* **Docs**: PRD (PRD-RNF-04), FDD (Seção 2, Seção 4.2), RFC (Seção 1, Seção 3 item 3) e ADR-005 documentam com precisão o loop contínuo de 2s e a latência de entrega inferior a 10s.

### 2.3. Políticas de Retry, Backoff e DLQ
* **Reunião**: Diego propõe em `[09:15]` um limite de 5 tentativas com backoff progressivo (1m, 5m, 30m, 2h, 12h) durando cerca de 15h. Propõe uma tabela física `webhook_dead_letter` em `[09:18]`.
* **Docs**: PRD (PRD-RNF-02), FDD (Seção 4.2, Seção 7), RFC (Seção 3 item 4) e ADR-002 detalham as 5 tentativas de forma idêntica. Também detalham o endpoint admin `/admin/webhooks/dead-letter/:id/replay` solicitado por Diego em `[09:18]` e restringido à role ADMIN por Sofia em `[09:35]`.

### 2.4. Segurança Criptográfica (HMAC-SHA256)
* **Reunião**: Sofia solicita HMAC-SHA256 em `[09:20]`, secret gerada de forma única por endpoint em `[09:21]` e suporte a rotação com grace period de 24 horas. Exige TLS/HTTPS estrito em `[09:23]`.
* **Docs**: PRD (PRD-FR-02, PRD-FR-03, PRD-FR-05, PRD-FR-07), FDD (Seção 3, Seção 5.2, Seção 12), RFC (Seção 3 item 1) e ADR-003 detalham o HMAC sobre o corpo brute do JSON, as chaves secretas únicas por endpoint, a validação de URLs seguras HTTPS no Zod, e o tempo de carência de 24h na rotação.

### 2.5. Garantia At-Least-Once e Idempotência
* **Reunião**: Diego estabelece a garantia de entrega de "pelo menos uma vez" com o header `X-Event-Id` contendo um UUID para deduplicação do lado do cliente em `[09:24]`.
* **Docs**: PRD (PRD-FR-08, PRD-RNF-03), FDD (Seção 3, Seção 7), RFC (Seção 1, Seção 5) e ADR-004 alinham-se estritamente sobre a impossibilidade de Exactly-Once de baixo custo e delegam a filtragem ao receptor com base no header.

### 2.6. Fora de Escopo Reais (Filtro de Ruído)
* **Reunião**: 
  * Email de alerta rejeitado por Larissa em `[09:37]`.
  * Dashboard visual rejeitado por Larissa em `[09:39]`.
  * Rate limiting de saída ficou sob observação e sem regras ativas em `[09:39]`.
  * Deleção histórica após 30 dias rejeitada na fase 1 por Diego em `[09:08]`.
* **Docs**: PRD (Seção 5.2) e FDD (Seção 3) listam estes 4 itens de forma idêntica como fora de escopo de implementação imediata, provando filtragem de ruído impecável.

---

## 3. Validação Cruzada: Código Existente vs. Design Docs

Mapeamento de referências estruturais da base de código do OMS para confirmar que nenhum arquivo físico ou classe citada é inexistente.

1. **`prisma/schema.prisma`**
   - **No Código**: Define enum `OrderStatus` com status (`PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`), tabela `User` com enum `UserRole` (`ADMIN`, `OPERATOR`), e modelo `Customer`.
   - **Nos Docs**: O FDD descreve perfeitamente que o CRUD utilizará esse schema do Prisma e o método transacional estenderá os modelos mapeando as entidades Webhooks em conformidade.
2. **`src/modules/orders/order.service.ts`**
   - **No Código**: Método `changeStatus` gerencia o ciclo transacional e utiliza o `tx` do Prisma `$transaction`.
   - **Nos Docs**: O FDD (Seção 9 item 2) cita os ganchos exatos e as linhas aproximadas da inserção transacional do outbox.
3. **`src/shared/errors/app-error.ts` e `src/shared/errors/index.ts`**
   - **No Código**: `AppError` estende a classe padrão `Error` do JS aceitando `message`, `statusCode` e `errorCode`.
   - **Nos Docs**: FDD (Seção 6, Seção 9) e ADR-006 citam de forma idêntica o reuso de `AppError` e herança direta para os erros customizados com o prefixo `WEBHOOK_`.
4. **`src/middlewares/auth.middleware.ts`**
   - **No Código**: Exporta funções de verificação de autenticação e o middleware de validação de role.
   - **Nos Docs**: FDD (Seção 5.4, Seção 9 item 4) e ADR-006 explicam o reuso do middleware para aplicar a restrição de role `UserRole.ADMIN` no replay da DLQ.
5. **`src/shared/logger/index.ts`**
   - **No Código**: Configura e expõe a instância global do Logger Pino.
   - **Nos Docs**: FDD (Seção 8, Seção 9 item 6) e ADR-006 alinham-se exigindo o reuso estrito dessa instância Pino no Worker isolado para manter a integridade dos logs.
6. **`src/routes/index.ts`**
   - **No Código**: Concentra as rotas das APIs do projeto.
   - **Nos Docs**: FDD (Seção 9 item 5) explica que modificaremos a árvore central para integrar as rotas do novo módulo de webhooks.

---

## 4. Conclusão da Revisão

Os artefatos encontram-se em estado **excepcional de consistência, conformidade e profundidade técnica**. Todas as metas de cobertura do Tracker e checklist do desafio foram superadas com folga. Nenhuma decisão ou regra contradiz a transcrição, e todas as referências ao código base correspondem a arquivos físicos, classes e constantes reais do repositório.
