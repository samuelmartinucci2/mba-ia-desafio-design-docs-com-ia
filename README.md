# Processo de Produção: Design Docs Gerados por IA

## Sobre o Desafio
Este desafio consistiu em assumir o papel de maestro de Inteligência Artificial para traduzir a transcrição literal de uma reunião técnica de alinhamento (`TRANSCRICAO.md`) e o código de uma aplicação em produção (um Order Management System - OMS) em um pacote de documentação técnica acionável e altamente profissional. A nova funcionalidade desenhada é um **Sistema de Webhooks de Notificação de Pedidos** (Outbound Webhooks) destinado a notificar parceiros corporativos B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) em tempo real sobre mudanças no status de seus pedidos.

O objetivo principal foi estruturar documentos complementares, consistentes entre si e livres de contradições ou alucinações, em diferentes níveis de abstração (PRD para produto/negócio, RFC para arquitetura, ADRs para decisões pontuais e FDD para especificação detalhada de implementação), garantindo 100% de rastreabilidade (grounding) entre as especificações e as discussões da reunião ou arquivos da base de código.

---

## Ferramentas de IA Utilizadas

- **Gemini CLI (alimentado por Gemini 1.5 Pro)**: Atuou como o assistente e agente principal de engenharia de software para ler o repositório, analisar a transcrição, gerar os rascunhos iniciais e refinar incrementalmente toda a documentação markdown.
- **Mecanismo de Auto-Edit do Gemini CLI**: Utilizado para realizar varreduras na codebase, assegurar que nenhum caminho de arquivo fictício fosse citado e aplicar revisões automáticas de formato markdown e ortografia.

---

## Workflow Adotado

Seguimos a **Ordem de Execução Sugerida** do desafio para garantir uma construção lógica incremental de baixo para cima (bottom-up):

1. **Setup de Branch**: Iniciamos criando a branch de trabalho `feature/webhook-design-docs` para isolar o desenvolvimento de documentação.
2. **Varredura e Exploração de Código**: Analisamos o arquivo `prisma/schema.prisma` e `src/modules/orders/order.service.ts` para entender os padrões de id (UUID), enums de status, tratamento transacional e erros personalizados (`AppError`).
3. **ADRs Primeiro**: Registramos as 6 decisões fundamentais da reunião técnica em arquivos separados de ADR. Decidir o esqueleto de infraestrutura primeiro facilitou o detalhamento dos documentos seguintes.
4. **RFC (Request for Comments)**: Consolidamos a proposta de arquitetura de alto nível, mapeando as alternativas descartadas na reunião (envio síncrono e brokers de mensageria adicionais) e conectando os links diretos para as 6 ADRs.
5. **FDD (Feature Design Document)**: Com a arquitetura proposta e as decisões tomadas, construímos a especificação profunda de implementação, desenhando contratos HTTP para 4 endpoints principais, gerando payloads detalhados de request/response em JSON e mapeando as conexões exatas com 6 arquivos da codebase existente.
6. **PRD (Product Requirement Document)**: Consolidamos o documento de negócios, definindo objetivos, métricas quantitativas, escopo (e o que ficou de fora), listando 10 requisitos funcionais claros e analisando riscos e mitigações.
7. **Tracker de Rastreabilidade**: Elaboramos o arquivo `docs/TRACKER.md` contendo um mapeamento cruzado rigoroso de 35 itens documentados vinculados aos timestamps exatos da reunião técnica ou caminhos de código.
8. **Documentação do Processo (README)**: Consolidamos este README final detalhando a jornada de co-criação humano-IA.

---

## Prompts Customizados

Abaixo estão dois prompts customizados relevantes que foram escritos para interagir e direcionar a inteligência artificial de forma cirúrgica na produção e revisão técnica:

### Prompt 1: Extração Estruturada de Decisões Arquiteturais (ADRs)
Este prompt foi desenvolvido para varrer a transcrição e extrair decisões técnicas precisas no padrão de formato MADR, mitigando alucinações de infraestrutura inexistente.
```markdown
Você é um Arquiteto de Software Sênior. Analise o arquivo TRANSCRICAO.md e a base de código do projeto.
Identifique as decisões técnicas chaves discutidas pelo time (Larissa, Diego, Bruno, Sofia, Marcos).
Gere de 5 a 8 Architecture Decision Records (ADRs) na pasta docs/adrs/ nomeados no formato:
'ADR-NNN-titulo-em-kebab-case.md'.

Para cada ADR, use obrigatoriamente a estrutura:
- Status
- Contexto (explicando as dores e a necessidade)
- Decisão (explicando detalhadamente a escolha técnica feita)
- Alternativas Consideradas (mencione pelo menos 1 alternativa real discutida ou plausível que foi descartada e o porquê)
- Consequências (liste prós e contras usando trade-offs claros)

Garanta que pelo menos um dos ADRs referencie explicitamente caminhos de arquivos ou classes existentes da nossa base de código em src/. Não invente soluções de infraestrutura adicionais que o time recusou (como Redis Cluster ou RabbitMQ).
```

### Prompt 2: Alinhamento de Contratos de API e Código Existente (FDD)
Este prompt direcionou a geração da seção de integração técnica do FDD com o ecossistema existente, assegurando grounding estrito de código.
```markdown
Como Engenheiro de Software Principal, escreva a especificação técnica de implementação (FDD) do Sistema de Webhooks.
O documento deve conter caminhos físicos reais existentes na nossa codebase em src/ e prisma/.
Vá além do genérico:
1. Nomeie pelo menos 4 arquivos reais e explique de forma acionável em qual linha ou método o webhook se integrará (ex: como o método changeStatus de src/modules/orders/order.service.ts será estendido usando a transação tx do Prisma).
2. Detalhe como usaremos as classes de erros existentes herdando de AppError em src/shared/errors/app-error.ts e como o middleware global de erros capturará essas exceções.
3. Desenhe os payloads JSON completos (Request e Response) para os endpoints: Cadastro de Webhooks, Rotação de Secrets com grace period, Histórico de Deliveries e Replay manual da DLQ.
4. Use o prefixo WEBHOOK_* para todos os códigos de erros da matriz.
```

---

## Iterações e Ajustes

Durante o processo de design dos documentos com a IA, realizamos **3 iterações principais** de revisão crítica e refinamento para eliminar inconsistências e superficialidades:

- **Ajuste 1: Correção do Prefixo de Erros na Matriz (FDD)**: No rascunho inicial gerado pela IA para o FDD, os códigos de erros técnicos da matriz foram listados no formato genérico `ERR_WEBHOOK_*` ou `ERR_INVALID_URL`. Corrigimos ativamente o comportamento lembrando a IA de que, conforme a reunião técnica (`[09:28] Bruno` e `[09:29] Larissa`), todos os erros devem seguir o padrão corporativo da codebase utilizando o prefixo unificado `WEBHOOK_` (ex: `WEBHOOK_INVALID_URL`, `WEBHOOK_NOT_FOUND`). A IA aplicou o ajuste de forma consistente nos documentos finais.
- **Ajuste 2: Rastreabilidade e Grace Period da Rotação de Secrets**: A primeira versão do PRD e da ADR de segurança geradas omitiram o período de carência (grace period) de 24 horas para rotação de secrets, tratando a rotação de forma instantânea. Intervimos apontando que isso violava a restrição levantada pela Engenheira de Segurança (Sofia em `[09:21] Sofia`), de que a secret antiga deve continuar válida por 24 horas em paralelo para evitar queda de produção nos sistemas parceiros. O fluxo foi corrigido no PRD, FDD e ADR correspondente, incluindo exemplos de timestamps de expiração no response payload.
- **Ajuste 3: Alinhamento de Atomicidade e Rollback de Transação**: A IA havia proposto de início que o webhook fosse disparado via evento pub/sub assíncrono interno após o commit do banco. Corrigimos essa lógica de implementação no FDD e na ADR-001 para refletir fielmente o acordo técnico da reunião (`[09:40] Bruno` e `[09:41] Diego`): a inserção na tabela `webhook_outbox` deve ocorrer **dentro da mesma transação SQL** do banco de dados MySQL para que, em caso de falha física na gravação do outbox, ocorra rollback total e consistente da alteração do pedido.

---

## Como Navegar a Entrega

Todos os arquivos gerados estão contidos no diretório raiz e na pasta `docs/`. Abaixo está o mapeamento dos caminhos físicos dos arquivos e a ordem lógica sugerida para leitura técnica:

```
. (raiz)
├── README.md                                  <-- [PASSO 1: Este documento de jornada humano-IA]
└── docs/
    ├── PRD.md                                 <-- [PASSO 2: Visão de Produto e Requisitos Funcionais]
    ├── RFC.md                                 <-- [PASSO 3: Proposta Arquitetural Geral e Trade-offs]
    ├── TRACKER.md                             <-- [PASSO 7: Rastreabilidade cruzada total e Auditoria]
    ├── FDD.md                                 <-- [PASSO 5: Desenho Técnico Detalhado, Contratos e Integração]
    └── adrs/                                  <-- [PASSO 4: Decisões Pontuais de Arquitetura (ADRs)]
        ├── ADR-001-padrao-outbox-no-mysql.md
        ├── ADR-002-politica-de-retry-com-backoff-e-dlq.md
        ├── ADR-003-autenticacao-hmac-sha256-com-secret-por-endpoint.md
        ├── ADR-004-garantia-at-least-once-com-x-event-id.md
        ├── ADR-005-worker-em-processo-separado-em-polling.md
        └── ADR-006-reuso-de-padroes-existentes-do-projeto.md
```

### Ordem de Leitura Recomendada:
1. **`README.md` (Este arquivo)**: Entenda o fluxo e as interações humano-IA que geraram a solução.
2. **`docs/PRD.md`**: Compreenda a dor de negócios das parceiras (Atlas, Max, Nova Cargo), as métricas de latência < 10s e os 10 requisitos funcionais chave.
3. **`docs/RFC.md`**: Veja a arquitetura macro do sistema proposta e as razões por que envio síncrono e brokers pesados adicionais foram descartados.
4. **`docs/adrs/`**: Explore em detalhe o embasamento e prós/contras de cada escolha técnica individual (Outbox, Retry, HMAC, At-Least-Once, Polling e Reuso).
5. **`docs/FDD.md`**: Analise a especificação técnica acionável para o time de desenvolvimento iniciar o código (contratos JSON, códigos de erro e integração na transação do `OrderService`).
6. **`docs/TRACKER.md`**: Valide e audite as pontes entre cada linha dos documentos técnicos e os timestamps originais da transcrição ou arquivos físicos existentes no projeto.
