# Tracker de Rastreabilidade

O Tracker é uma tabela de referência cruzada que mapeia cada item documentado (requisitos, decisões, restrições e trade-offs) diretamente à sua fonte original, seja ela um trecho da reunião gravada em `TRANSCRICAO.md` ou um arquivo físico presente na codebase do projeto em `src/` ou `prisma/`.

---

## Tabela de Rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| **PRD-FR-01** | `docs/PRD.md` | Requisito Funcional | Cadastro de webhook via POST com URL e eventos. | TRANSCRICAO | `[09:31] Marcos` |
| **PRD-FR-02** | `docs/PRD.md` | Requisito Funcional | Validação estrita de protocolo seguro HTTPS para URL do webhook. | TRANSCRICAO | `[09:23] Sofia` |
| **PRD-FR-03** | `docs/PRD.md` | Requisito Funcional | Geração de secret aleatória de alta entropia devolvida apenas na criação. | TRANSCRICAO | `[09:31] Marcos` |
| **PRD-FR-04** | `docs/PRD.md` | Requisito Funcional | CRUD de gerenciamento de webhooks do cliente (GET, PATCH, DELETE). | TRANSCRICAO | `[09:33] Bruno` |
| **PRD-FR-05** | `docs/PRD.md` | Requisito Funcional | Rotação de secret com grace period de 24 horas para a secret anterior. | TRANSCRICAO | `[09:21] Sofia` |
| **PRD-FR-06** | `docs/PRD.md` | Requisito Funcional | Filtragem de eventos transacionais de status na inserção do outbox para economizar espaço. | TRANSCRICAO | `[09:33] Bruno` |
| **PRD-FR-07** | `docs/PRD.md` | Requisito Funcional | Assinatura criptográfica HMAC-SHA256 no header `X-Signature` sobre o body. | TRANSCRICAO | `[09:20] Sofia` |
| **PRD-FR-08** | `docs/PRD.md` | Requisito Funcional | Cabeçalhos HTTP de controle (`X-Event-Id`, `X-Timestamp`, `X-Webhook-Id`). | TRANSCRICAO | `[09:25] Diego` e `[09:44] Sofia` |
| **PRD-FR-09** | `docs/PRD.md` | Requisito Funcional | Histórico de entregas (últimas 100 tentativas) no endpoint `GET /webhooks/:id/deliveries`. | TRANSCRICAO | `[09:34] Marcos` |
| **PRD-FR-10** | `docs/PRD.md` | Requisito Funcional | Replay manual de DLQ restrito a ADMIN com log de auditoria. | TRANSCRICAO | `[09:18] Diego` e `[09:35] Sofia` |
| **PRD-RNF-01** | `docs/PRD.md` | Requisito Não Funcional | Desacoplamento de rede do envio físico e atomicidade local via Outbox. | TRANSCRICAO | `[09:06] Diego` |
| **PRD-RNF-02** | `docs/PRD.md` | Requisito Não Funcional | Tolerância a falhas com retry de 5 tentativas em janela de 15h. | TRANSCRICAO | `[09:15] Diego` e `[09:17] Diego` |
| **PRD-RNF-03** | `docs/PRD.md` | Requisito Não Funcional | Teto limite físico para tamanho do payload fixado em 64KB. | TRANSCRICAO | `[09:23] Sofia` e `[09:24] Diego` |
| **PRD-RNF-04** | `docs/PRD.md` | Requisito Não Funcional | Timeout máximo de chamada HTTP externa regulado em 10 segundos. | TRANSCRICAO | `[09:42] Diego` |
| **PRD-RNF-05** | `docs/PRD.md` | Requisito Não Funcional | Controle de acesso a replays restrito à role ADMIN e verificado via JWT. | TRANSCRICAO | `[09:35] Sofia` |
| **ADR-001** | `docs/adrs/ADR-001-padrao-outbox-no-mysql.md` | Decisão Arquitetural | Adoção do padrão Transactional Outbox persistido no MySQL do projeto. | TRANSCRICAO | `[09:06] Diego` |
| **ADR-002** | `docs/adrs/ADR-002-politica-de-retry-com-backoff-e-dlq.md` | Decisão Arquitetural | Política de retry progressivo (1m/5m/30m/2h/12h) e isolamento físico em DLQ. | TRANSCRICAO | `[09:15] Diego` e `[09:18] Diego` |
| **ADR-003** | `docs/adrs/ADR-003-autenticacao-hmac-sha256-com-secret-por-endpoint.md` | Decisão Arquitetural | Criptografia HMAC-SHA256 e rotação autônoma de secrets com carência de 24h. | TRANSCRICAO | `[09:20] Sofia` e `[09:21] Sofia` |
| **ADR-004** | `docs/adrs/ADR-004-garantia-at-least-once-com-x-event-id.md` | Decisão Arquitetural | Garantia de entrega At-Least-Once com tratamento de idempotência via `X-Event-Id`. | TRANSCRICAO | `[09:24] Diego` |
| **ADR-005** | `docs/adrs/ADR-005-worker-em-processo-separado-em-polling.md` | Decisão Arquitetural | Worker em processo independente (`npm run worker`) rodando polling a cada 2 segundos. | TRANSCRICAO | `[09:09] Diego` e `[09:11] Diego` |
| **ADR-006** | `docs/adrs/ADR-006-reuso-de-padroes-existentes-do-projeto.md` | Decisão Arquitetural | Alinhamento do novo módulo com estruturas e classes existentes do projeto. | TRANSCRICAO | `[09:27] Bruno` e `[09:30] Larissa` |
| **RFC-ALT-01** | `docs/RFC.md` | Alternativa Rejeitada | Rejeição de requisições síncronas de webhooks direto no fluxo da API. | TRANSCRICAO | `[09:04] Bruno` |
| **RFC-ALT-02** | `docs/RFC.md` | Alternativa Rejeitada | Rejeição de brokers de mensageria adicionais (Redis Streams/RabbitMQ) para evitar custos. | TRANSCRICAO | `[09:07] Diego` e `[09:07] Larissa` |
| **RFC-OPEN-01** | `docs/RFC.md` | Questão em Aberto | Avaliar necessidade futura de Throttling e Rate Limiting no Worker de saída. | TRANSCRICAO | `[09:38] Diego` e `[09:39] Larissa` |
| **RFC-OPEN-02** | `docs/RFC.md` | Questão em Aberto | Estratégia de concorrência distribuída versus ordenação implícita (Single-worker). | TRANSCRICAO | `[09:12] Diego` e `[09:13] Diego` |
| **OUT-01** | `docs/PRD.md` | Fora de Escopo | Envio automático de alertas via e-mail corporativo em caso de falhas consecutivas. | TRANSCRICAO | `[09:37] Larissa` |
| **OUT-02** | `docs/PRD.md` | Fora de Escopo | Dashboard/Painel visual de controle e logs do webhook no Frontend. | TRANSCRICAO | `[09:39] Larissa` |
| **OUT-03** | `docs/PRD.md` | Fora de Escopo | Arquivamento histórico em massa ou deleção de logs antigos do banco (30 dias). | TRANSCRICAO | `[09:08] Diego` |
| **FDD-CONTRATO-01** | `docs/FDD.md` | Especificação de API | Formato do payload leve de notificação contendo apenas dados gerais do pedido (sem itens). | TRANSCRICAO | `[09:43] Diego` e `[09:44] Bruno` |
| **FDD-INTEG-01** | `docs/FDD.md` | Integração de Código | Persistência atômica da outbox dentro da transação em `OrderService.changeStatus`. | CODIGO | `src/modules/orders/order.service.ts` |
| **FDD-INTEG-02** | `docs/FDD.md` | Integração de Código | Extensão do banco de dados e modelagem de modelos Prisma associados a UUID. | CODIGO | `prisma/schema.prisma` |
| **FDD-INTEG-03** | `docs/FDD.md` | Integração de Código | Tratamento central de erros de negócio através da herança da classe customizada `AppError`. | CODIGO | `src/shared/errors/app-error.ts` |
| **FDD-INTEG-04** | `docs/FDD.md` | Integração de Código | Controle de acesso a rotas administrativas usando `requireRole` e o middleware JWT. | CODIGO | `src/middlewares/auth.middleware.ts` |
| **FDD-INTEG-05** | `docs/FDD.md` | Integração de Código | Centralização do registro de eventos de polling e envio de webhooks usando o Pino Logger. | CODIGO | `src/shared/logger/index.ts` |
| **FDD-INTEG-06** | `docs/FDD.md` | Integração de Código | Registro das novas rotas de webhooks do cliente e rotas de admin na árvore central de rotas. | CODIGO | `src/routes/index.ts` |

---

## Verificação das Regras de Cobertura
- **Total de Itens Identificáveis**: 35 itens mapeados.
- **Cobertura Geral**: 100% dos itens chave documentados em PRD, RFC, FDD e ADRs possuem registro correspondente no Tracker, superando com facilidade a meta mínima de **80%**.
- **Fonte = TRANSCRICAO**: 29 de 35 linhas têm Fonte = `TRANSCRICAO` com timestamps cronológicos reais e válidos. Isso representa **82.8%** do total (meta mínima: **70%**).
- **Fonte = CODIGO**: 6 linhas têm Fonte = `CODIGO` referenciando caminhos de arquivos físicos reais existentes no repositório (meta mínima: **5 linhas**).
