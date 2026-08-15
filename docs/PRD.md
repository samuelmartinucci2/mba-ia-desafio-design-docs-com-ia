# PRD: Product Requirement Document - Sistema de Webhooks de Notificação de Pedidos

## 1. Resumo e Contexto da Feature
O Order Management System (OMS) atualmente opera em um fluxo transacional robusto para gerenciar o ciclo de vida dos pedidos. No entanto, não há nenhuma via nativa para notificar sistemas externos sobre mudanças no estado dessas entidades. O **Sistema de Webhooks de Notificação de Pedidos** é uma nova funcionalidade que permitirá que parceiros corporativos B2B do ecossistema recebam atualizações push automáticas em "tempo real" (delay abaixo de 10 segundos) diretamente em seus servidores de destino (Endpoints HTTP) sempre que um pedido sofrer transição de status (ex: de `PENDING` para `PAID` ou `SHIPPED`).

---

## 2. Problema e Motivação
Grandes clientes B2B (como Atlas Comercial, MaxDistribuição e Nova Cargo) necessitam de alta sincronia operacional para processar o despacho e a entrega de mercadorias. Atualmente, para manter seus sistemas locais sincronizados com o OMS, esses parceiros realizam consultas exaustivas e frequentes (polling de alta frequência) na nossa API pública (`GET /orders`). 

Essa abordagem gera:
1. **Sobrecarga Inútil na Infraestrutura**: Milhares de requisições GET retornam dados idênticos sem alteração, gerando consumo excessivo de I/O de banco de dados e CPU na nossa API principal.
2. **Latência de Integração**: Há uma janela de atraso inevitável entre a alteração real do pedido e a leitura do cliente.
3. **Ameaça de Churn**: Clientes estratégicos corporativos (como a Atlas Comercial) expressaram que a falta deste mecanismo "tempo real" compromete sua eficiência e cogitam migrar para soluções concorrentes caso o recurso não seja entregue até o término do trimestre corrente.

---

## 3. Público-Alvo e Cenários de Uso
* **Público-Alvo**: Engenheiros de integração, administradores de TI e desenvolvedores dos parceiros corporativos B2B integrados à nossa plataforma.
* **Cenário de Uso 1 (Atlas Comercial)**: 
  * O operador do OMS altera o status do pedido `ORD-000123` para `SHIPPED`.
  * Automaticamente, o OMS envia um disparo HTTP POST contendo o payload do pedido para o endpoint HTTPS registrado pela Atlas.
  * O sistema da Atlas recebe e dispara a roteirização física do transporte do pedido em menos de 10 segundos.
* **Cenário de Uso 2 (MaxDistribuição)**:
  * A MaxDistribuição quer apenas receber notificações quando o pedido for pago (`PAID`) ou cancelado (`CANCELLED`). 
  * Eles configuram seu webhook filtrando apenas por estes eventos, ignorando as atualizações intermediárias de logística.

---

## 4. Objetivos e Métricas de Sucesso

| Objetivo | Métrica de Sucesso | Meta Quantitativa |
| --- | --- | --- |
| **Garantir Entrega Rápida** | Tempo decorrido entre a transição do pedido e o recebimento pelo parceiro | **Latência < 10 segundos** no percentil 95 (p95) |
| **Mitigar Sobrecarga de API** | Redução do volume de requisições `GET /orders` ineficientes dos parceiros integrados | **Redução de no mínimo 80%** das chamadas repetitivas de polling desses clientes |
| **Alta Disponibilidade e Resiliência**| Taxa de sucesso geral na entrega de eventos válidos | **Sucesso > 99.9%** de entregas efetivadas (incluindo retries automatizados) |

---

## 5. Escopo

### 5.1. Em Escopo (In-Scope):
- Configuração de Webhooks por cliente (URLs HTTPS, seleção de eventos por Status).
- Geração de chaves secretas exclusivas por webhook de forma automatizada.
- Assinatura criptográfica HMAC-SHA256 para comprovar autoria e integridade.
- Garantia de entrega At-Least-Once (pelo menos uma vez).
- Header de idempotência `X-Event-Id` para o cliente deduplicar requisições de retry.
- Histórico de auditoria das últimas 100 tentativas de envio por endpoint.
- Rotação autônoma de secrets com carência de segurança (grace period) de 24 horas.
- Retries automáticos com backoff exponencial (5 tentativas em até 15 horas).
- Isolamento de mensagens mortas em fila de descarte (DLQ) física em banco.
- Interface de API restrita (perfil ADMIN) para disparar novamente (replay) mensagens mortas da DLQ.

### 5.2. Fora de Escopo (Out-of-Scope):
- **Painel Visual de Configuração (Dashboard Frontend)**: Esta fase focará puramente no backend e fornecimento de endpoints de API para integração. O painel visual é um projeto delegado ao time de frontend para o trimestre subsequente.
- **Notificação Automática por E-mail de Falhas Críticas**: Notificar o cliente via e-mail ou Slack quando seu endpoint de webhook começar a falhar e for movido para a DLQ está postergado para a Fase 2.
- **Throttling de Envio de Saída**: Mecanismos ativos de limitação e enfileiramento de taxa máxima de chamadas de saída para evitar DoS nos clientes. O comportamento será monitorado nesta fase e implementado se necessário.
- **Desativação Automática de Webhook Quebrado**: Desativar o webhook automaticamente caso ele apresente 100% de erro nas últimas 24 horas.

---

## 6. Requisitos Funcionais (Mínimo de 8)

1. **PRD-FR-01 (Cadastro de Webhook)**: O sistema deve permitir que clientes autenticados cadastrem endpoints de webhook fornecendo uma URL e uma lista de eventos (status de pedidos) que desejam assinar.
2. **PRD-FR-02 (Validação HTTPS)**: O sistema deve validar e exigir estritamente que a URL cadastrada utilize o protocolo seguro HTTPS (bloquear cadastros HTTP).
3. **PRD-FR-03 (Geração Automática de Secret)**: Durante a criação do webhook, o sistema deve gerar automaticamente uma chave criptográfica secreta única e de alta entropia, devolvendo-a uma única vez ao cliente para validação futura.
4. **PRD-FR-04 (Consulta de Configurações)**: O sistema deve permitir que o cliente liste, consulte detalhes, ative/desative, atualize ou remova suas configurações de webhooks cadastradas.
5. **PRD-FR-05 (Rotação Ativa de Secrets)**: O cliente deve conseguir rotacionar a secret do webhook. Quando rotacionada, a secret antiga deve permanecer válida por exatamente 24 horas em paralelo (grace period) para evitar quedas no ambiente do parceiro.
6. **PRD-FR-06 (Filtro de Eventos Transacionais)**: O sistema deve filtrar as notificações no momento da mudança de status do pedido. Se um pedido mudar de status e o cliente não tiver assinado aquele status específico, o evento não deve ser gravado na outbox.
7. **PRD-FR-07 (Assinatura de Integridade HMAC)**: Cada envio efetuado pelo Worker de webhooks deve conter o cabeçalho `X-Signature` contendo o hash HMAC-SHA256 do corpo do request gerado com a respectiva secret ativa do webhook.
8. **PRD-FR-08 (Cabeçalhos de Idempotência)**: Cada chamada HTTP de webhook deve possuir os cabeçalhos de controle: `X-Event-Id` (UUID do evento para dedup), `X-Timestamp` (data/hora do envio atual para prevenir replay attacks) e `X-Webhook-Id` (ID de cadastro do webhook receptor).
9. **PRD-FR-09 (Histórico de Auditoria)**: O sistema deve expor um histórico legível contendo as últimas 100 tentativas de envio realizadas por webhook, especificando data de criação, tentativa, sucesso, tempo de resposta em milissegundos e código HTTP recebido.
10. **PRD-FR-10 (Replay Administrativo de DLQ)**: O sistema deve expor um endpoint restrito para usuários com perfil `ADMIN` para acionar manualmente o reenvio (replay) de um evento localizado na DLQ, registrando o log de auditoria da ação.

---

## 7. Requisitos Não Funcionais (RNF)

1. **PRD-RNF-01 (Desacoplamento e Atomicidade)**: O registro de intenção de envio do webhook deve ser transacional e atômico com a atualização do pedido. No entanto, o envio físico da requisição HTTP de rede deve ocorrer de forma 100% assíncrona, desacoplada do fluxo HTTP principal de API.
2. **PRD-RNF-02 (Tolerância a Falhas e Janela de Retry)**: O sistema deve reexecutar envios falhos em uma janela progressiva de 5 tentativas espaçadas (1m, 5m, 30m, 2h, 12h) antes de desistir e arquivar na DLQ.
3. **PRD-RNF-03 (Teto Limite de Payload)**: O tamanho total de um payload de evento gerado para envio de webhook não deve ultrapassar **64KB**. Caso exceda, o sistema deve abortar o envio registrando o erro na DLQ.
4. **PRD-RNF-04 (Timeout do Disparo)**: O timeout máximo tolerado para a resposta HTTP do servidor do parceiro deve ser configurado rigidamente em **10 segundos** para evitar vazamento de sockets e bloqueio do Worker.
5. **PRD-RNF-05 (Segurança de Acesso de Administração)**: O controle de acesso ao reprocessamento da DLQ deve exigir autenticação robusta via JWT e validação de perfil `ADMIN`.

---

## 8. Decisões e Trade-offs Principais
* **Transactional Outbox vs Event Broker dedicado (Redis/RabbitMQ)**: Optamos por implementar o padrão Outbox armazenado no banco MySQL atual. *Trade-off*: Evitamos custos imediatos e complexidade operacional de subir nova infraestrutura em um time enxuto, aceitando o trade-off de um overhead controlado de escrita de transações adicionais no MySQL existente.
* **Garantia At-Least-Once vs Exactly-Once**: Garantiremos entrega de "pelo menos uma vez". *Trade-off*: Fornecemos resiliência máxima na entrega do evento, delegando de forma explícita ao cliente o dever de ignorar requisições redundantes idênticas por meio do header de idempotência `X-Event-Id`.

---

## 9. Dependências
- **MySQL e Prisma Client**: Dependência direta de persistência para as novas tabelas e garantia transacional SQL.
- **Zod e AppError**: Reuso obrigatório para a consistência e conformidade com os padrões de tratamento de erro do projeto.
- **Middleware requireRole e JWT**: Dependência de segurança para autenticar e autorizar acessos administrativos.

---

## 10. Riscos e Mitigação (Mínimo de 2)

### Risco 1: Bloqueio do Event Loop da API Principal por Webhooks Lentos
* **Probabilidade**: Baixa
* **Impacto**: Crítico
* **Mitigação**: O envio físico do webhook é processado por um Worker rodando em um processo Node.js totalmente isolado (`src/worker.ts`), desacoplado da API de vendas principal (`src/server.ts`).

### Risco 2: Ataques de Replay de Rede em Clientes Receptoras
* **Probabilidade**: Média
* **Impacto**: Alto
* **Mitigação**: Enviamos a assinatura criptográfica e segura `X-Signature` (HMAC-SHA256) atrelada ao corpo bruto e o cabeçalho `X-Timestamp`. Os clientes são instruídos a ignorar pacotes antigos cujo timestamp possua divergência severa (ex: mais de 5 minutos de atraso).

---

## 11. Critérios de Aceitação
- Uma transição de status de pedido bem-sucedida deve gerar um evento na outbox se o cliente assinar o evento.
- Se a atualização do pedido sofrer rollback, nenhum registro deve ser criado na outbox.
- O Worker deve processar eventos em lotes a cada 2 segundos, emitindo logs detalhados do Pino para sucesso ou falha.
- Uma chamada HTTP mal-sucedida com timeout de 10s deve acionar retry e respeitar as janelas exponenciais até cair na DLQ.
- Rotas restritas para ADMIN devem recusar chamadas de OPERATORS com código `403 Forbidden`.

---

## 12. Estratégia de Testes e Validação
- **Testes Unitários**: Cobertura das classes de validação Zod (HTTPS obrigatório, limites de payload de 64KB) e funções puras de cálculo de HMAC-SHA256.
- **Testes de Integração**: Simular a transição de pedido e verificar a inserção correta e transacional na tabela de outbox. Garantir o comportamento de rollback simulando erros forçados.
- **Testes de Resiliência (Mock HTTP)**: Rodar o Worker com servidores mock instáveis (simulando 504 Gateway Timeout, Bad Request e lentidões acima de 10 segundos) e validar as transições para status `FAILED` com backoff e posterior inserção na DLQ.
- **Testes de Segurança**: Validar assinaturas HMAC forjadas para atestar que requisições adulteradas são detectadas e rejeitadas pelo cliente. Testar controle de roles (ADMIN vs OPERATOR) para o replay da DLQ.
