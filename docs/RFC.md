# RFC: Sistema de Webhooks de Notificação de Pedidos

## Metadados
- **Autor**: Bruno (Engenheiro Pleno, Time de Pedidos) & Diego (Engenheiro Sênior, Time de Plataforma)
- **Status**: Sob Revisão
- **Data**: 15 de Agosto de 2026
- **Revisores**: 
  - Larissa (Tech Lead)
  - Marcos (Product Manager)
  - Sofia (Engenheira de Segurança)

---

## 1. Resumo Executivo (TL;DR)
Esta RFC propõe o projeto arquitetural de um **Sistema de Webhooks de Notificação de Pedidos (Outbound Webhooks)**. A feature permitirá que clientes B2B (como Atlas Comercial, MaxDistribuição e Nova Cargo) recebam notificações HTTP em tempo real (latência inferior a 10 segundos) sempre que o status de seus pedidos for alterado no nosso Order Management System (OMS). A proposta baseia-se no padrão **Transactional Outbox** implementado sobre o MySQL existente e executado por um **Worker separado por Polling**, garantindo confiabilidade atômica (at-least-once), segurança criptográfica via assinaturas **HMAC-SHA256**, tolerância a falhas com **Backoff Exponencial** e isolamento operacional através de uma **Dead Letter Queue (DLQ)**.

---

## 2. Contexto e Problema
Atualmente, nosso OMS não oferece nenhuma via de notificação push ou streaming para eventos externos. Clientes corporativos B2B precisam consultar de forma repetitiva (polling) o endpoint `GET /orders` para identificar se houve mudança de status em seus pedidos. Esta abordagem acarreta sérios problemas de performance, latência de integração e sobrecarga desnecessária na nossa infraestrutura de API.

Para solucionar esse problema e reter clientes críticos (como a Atlas Comercial), precisamos fornecer um mecanismo escalável e seguro para empurrar (push) as atualizações de status de pedidos em "tempo real" (delay aceitável inferior a 10 segundos) diretamente para endpoints HTTP fornecidos por esses clientes.

---

## 3. Proposta Técnica (Visão Geral)
A arquitetura proposta baseia-se no desacoplamento assíncrono dos fluxos de escrita de pedidos e de entrega de notificações, visando resiliência e isolamento.

```
+--------------------------------------------------------+
|                      API PRINCIPAL                     |
|                                                        |
|  [POST /orders/change-status]                          |
|               |                                        |
|               v                                        |
|     +-------------------+                              |
|     |  Transação MySQL  |                              |
|     |  (order.service)  |                              |
|     |  +-------------+  |                              |
|     |  | Update Order|  |                              |
|     |  +-------------+  |                              |
|     |  |  Write      |  |                              |
|     |  |  Outbox     |  |                              |
|     |  +-------------+  |                              |
|     +---------|---------+                              |
+---------------|----------------------------------------+
                | (Persistido no MySQL)
                v
      +--------------------+
      |  Tabela Outbox     |
      +--------------------+
                ^
                | (Polling a cada 2s)
+---------------|----------------------------------------+
|               |        WORKER SEPARADO                 |
|  +------------+-------------+                          |
|  | webhook.worker.ts        |                          |
|  |                          |                          |
|  |  1. Busca pendentes      |                          |
|  |  2. Assina c/ HMAC       |                          |
|  |  3. Dispara HTTP POST    |-- (Sucesso) -> [Removido/|
|  +------------+-------------+                 Finalizado]
|               |                                        |
|               +-- (5 Falhas) -> [Tabela Dead Letter]   |
+--------------------------------------|-----------------+
                                       v
                             [DLQ: webhook_dead_letter]
                                       ^
                                       | (Replay Manual)
                             [POST /admin/webhooks/...]
```

### Componentes Principais:
1. **Configuração de Webhooks**: Endpoints de CRUD (`POST`, `GET`, `PATCH`, `DELETE` em `/webhooks`) para os clientes cadastrarem e gerenciarem suas URLs de destino, chaves secretas (HMAC) e os eventos (status dos pedidos) que desejam assinar.
2. **Transactional Outbox**: Ao invés de disparar requisições HTTP de forma síncrona, a API principal do OMS salvará a intenção da notificação (snapshot do payload) em uma tabela `webhook_outbox` como parte da mesma transação SQL que altera o status do pedido em `src/modules/orders/order.service.ts`.
3. **Background Worker**: Um processo Node.js independente (`src/worker.ts`), executando em um loop de polling de 2 segundos. Ele busca lotes de eventos elegíveis na outbox, calcula as assinaturas HMAC-SHA256, executa as requisições HTTP e atualiza o estado de processamento.
4. **Resiliência e DLQ**: Tentativas malsucedidas acionam uma escala de retry exponencial com até 5 tentativas ao longo de 15 horas. Casos de falha irreversível são movidos para a tabela `webhook_dead_letter` (DLQ) para reprocessamento manual via endpoint restrito de administração.

---

## 4. Alternativas Consideradas

Durante o alinhamento técnico, analisamos e descartamos duas abordagens alternativas:

### Alternativa 1: Envio Síncrono direto no OrderService
Consiste em realizar as requisições de rede HTTP diretamente dentro do método `changeStatus` de `OrderService` antes de consolidar a transação.
* **Por que foi descartada?** Sobrecarriga drasticamente a transação SQL principal. Clientes lentos travariam a mudança de status de outros pedidos e esgotariam o pool de conexões do MySQL. Em caso de queda do cliente, não conseguiríamos dar rollback na atualização do pedido ou perderíamos a notificação sem capacidade de reenvio assíncrono seguro.

### Alternativa 2: Mensageria Dedicada com Redis Streams ou RabbitMQ
Consiste em utilizar um broker de mensageria externo dedicado para enfileirar as notificações de webhooks e consumi-las de forma puramente reativa por meio de assinaturas de eventos.
* **Por que foi descartada?** O projeto atual não possui essa infraestrutura provisionada. Subir e gerenciar um Redis Cluster ou um servidor RabbitMQ apenas para esta feature introduziria custos financeiros e complexidade operacional desproporcionais (overengineering) para o tamanho atual da equipe e escopo inicial dos clientes B2B. O banco MySQL existente possui suporte transacional robusto e capacidade de sobra para gerenciar a tabela outbox.

---

## 5. Decisões Relacionadas (ADRs)
Para detalhes fundamentais sobre cada escolha arquitetural, consulte os Architecture Decision Records vinculados abaixo:

* [ADR-001: Padrão Outbox no MySQL](adrs/ADR-001-padrao-outbox-no-mysql.md) - Detalha a escolha do padrão Transactional Outbox transacionado e a modelagem no banco.
* [ADR-002: Política de Retry com Backoff e DLQ](adrs/ADR-002-politica-de-retry-com-backoff-e-dlq.md) - Especifica as janelas de reenvio de até 15 horas e a estrutura de Dead Letter Queue.
* [ADR-003: Autenticação HMAC-SHA256 com Secret por Endpoint](adrs/ADR-003-autenticacao-hmac-sha256-com-secret-por-endpoint.md) - Descreve a camada de assinatura criptográfica de integridade de dados e grace period de rotação.
* [ADR-004: Garantia At-Least-Once com X-Event-Id](adrs/ADR-004-garantia-at-least-once-com-x-event-id.md) - Regula as diretrizes de entrega persistente e deduplicação no lado do cliente.
* [ADR-005: Worker em Processo Separado em Polling](adrs/ADR-005-worker-em-processo-separado-em-polling.md) - Justifica a execução assíncrona desacoplada da API principal e a latência de 2 segundos.
* [ADR-006: Reuso de Padrões Existentes do Projeto](adrs/ADR-006-reuso-de-padroes-existentes-do-projeto.md) - Aborda a conformidade técnica com o ecossistema existente (Pino, AppError, Zod, requireRole).

---

## 6. Questões em Aberto
Identificamos pontos de atenção técnica que foram levantados pela equipe na reunião técnica e que foram adiados ou classificados como "a observar":

1. **Rate Limiting de Saída (Outbound Rate Limit)**: Se um cliente sofrer um pico de transições de status de pedidos (ex: 50 pedidos por minuto), o Worker enviará 50 requisições simultâneas para o endpoint dele, agindo como um vetor involuntário de negação de serviço (DoS). Decidiu-se observar o comportamento em produção nesta fase inicial e implementar mecanismos de throttling ou particionamento no Worker apenas se necessário futuramente.
2. **Garantia de Ordering Global**: Múltiplos Workers escalados horizontalmente violam a cronologia exata de recebimento do histórico de status de um pedido do lado do cliente. Optou-se por rodar o Worker de forma estritamente isolada (single-worker) neste primeiro trimestre, limitando-o à ordenação implícita baseada no id cronológico, postergando soluções como locks pessimistas ou partições por ID para quando a escala exigir múltiplos trabalhadores.

---

## 7. Impacto e Riscos
* **Impacto no Banco de Dados**: A tabela de outbox crescerá de forma linear conforme o volume de vendas aumente. Para mitigar degradação de performance nas rotinas de busca, criaremos índices compostos específicos. Além disso, rotinas de arquivamento ou deleção de registros finalizados com mais de 30 dias serão planejadas em uma etapa posterior.
* **Segurança de Rede (Inundação de Conexões)**: Disparar requisições para servidores terceiros de forma assíncrona pode prender sockets do Worker caso os servidores dos clientes apresentem lentidão excessiva. Mitigaremos isso limitando rigidamente o timeout das requisições HTTP para **10 segundos** e gerenciando de forma restrita o tamanho do pool de agentes de conexões (keep-alive) do Worker.
