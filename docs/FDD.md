# FDD: Feature Design Document - Sistema de Webhooks de Notificação de Pedidos

## 1. Contexto Técnico e Motivação
Este documento detalha as especificações técnicas de baixo nível para a implementação do Sistema de Webhooks do Order Management System (OMS). A feature é necessária para permitir o envio assíncrono e confiável de eventos de mudança de status de pedidos para parceiros B2B corporativos (Atlas Comercial, MaxDistribuição, Nova Cargo). A implementação deve garantir o desacoplamento de rede, resiliência no envio, integridade criptográfica e total alinhamento com os padrões arquiteturais existentes na codebase.

---

## 2. Objetivos Técnicos
- Latência média de disparo inferior a 10 segundos (polling de 2 segundos).
- Taxa de sucesso de entrega visada superior a 99.9% através de retries automáticos.
- Zero impacto no tempo de resposta das requisições HTTP de alteração de status de pedido da API principal.
- Isolar completamente falhas de terceiros (timeouts, quedas de clientes) para impedir vazamento de recursos ou travamento de processos do OMS.

---

## 3. Escopo e Exclusões

### Em Escopo de Implementação:
- CRUD completo de gerenciamento de webhooks associados a clientes (Customers).
- Modelagem e migração de banco de dados (Prisma/MySQL) das tabelas `webhooks`, `webhook_outbox` e `webhook_dead_letter`.
- Lógica transacional de persistência de eventos no método `changeStatus` do `OrderService`.
- Worker executado em loop de 2s via script `npm run worker` (processo isolado).
- Assinatura criptográfica HMAC-SHA256 no header `X-Signature`.
- Cabeçalhos de rastreamento e idempotência (`X-Event-Id`, `X-Timestamp`, `X-Webhook-Id`).
- Rotação autônoma de secrets com carência ativa de 24 horas.
- Endpoint de histórico com limite das últimas 100 tentativas (`GET /webhooks/:id/deliveries`).
- Endpoint administrativo de Replay de DLQ (`POST /admin/webhooks/dead-letter/:id/replay`).

### Fora de Escopo nesta Fase (Postergado/Excluído):
- Interface visual (Frontend/Dashboard) para visualização ou configuração de webhooks.
- Notificação de falha via e-mail ou canais externos quando o webhook falha e vai para a DLQ.
- Rotina automatizada de desativação permanente de webhooks que acumulam erros frequentes.
- Throttling ou controle dinâmico de taxa de envio de saída (Outbound Rate Limiting).
- Rotina de arquivamento ou deleção histórica para dados de outbox finalizados após 30 dias.

---

## 4. Fluxos Detalhados

### 4.1. Fluxo de Criação do Evento na Outbox (API)
1. O operador realiza uma requisição HTTP para alterar o status do pedido (`PATCH /orders/:id/status`).
2. O `OrderService.changeStatus` abre uma transação no banco através do `this.prisma.$transaction`.
3. Valida a transição de status e atualiza a entidade do pedido.
4. Grava a alteração na tabela `order_status_history`.
5. Consulta os webhooks ativos do respectivo `customerId` que assinam aquele status específico.
6. Se houver webhooks elegíveis, renderiza o snapshot completo do payload em formato JSON.
7. Para cada webhook habilitado, insere uma linha correspondente na tabela `webhook_outbox` com status `PENDING`, `attempts = 0` e UUID próprio.
8. A transação realiza o commit no MySQL. Se houver qualquer falha em qualquer etapa (incluindo escrita do outbox), toda a transação sofre rollback.

### 4.2. Fluxo de Processamento pelo Worker
```
+------------------+
|   Loop 2s        |
+--------|---------+
         |
         v
+-----------------------------+
| Buscar no MySQL             |
| status = PENDING ou FAILED  |
| AND attempts < 5            |
| AND nextAttemptAt <= NOW()  |
+--------|--------------------+
         | (Se houver registros)
         v
+------------------------------------------+
| Para cada evento (paralelo controlado):  |
| 1. Atualiza status = PROCESSING          |
| 2. Recupera secret ativa do webhook      |
| 3. Stringifica o payload do outbox       |
| 4. Calcula assinatura HMAC-SHA256        |
| 5. Envia HTTP POST (timeout 10s)         |
+--------|---------------------------------+
         |
         +------------------+------------------+
         | (HTTP 2xx)       | (HTTP >=300, Timeout ou Rede)
         v                  v
+------------------+   +------------------------------------+
| Sucesso:         |   | Falha:                             |
| 1. status = DELI-|   | 1. Incrementa attempts             |
|    VERED         |   | 2. Se attempts < 5:                |
| 2. Registra o    |   |    - status = FAILED               |
|    delivery log  |   |    - nextAttemptAt = NOW() + back- |
| 3. Remove/Fina-  |   |      off (1m/5m/30m/2h/12h)         |
|    liza registro |   | 3. Se attempts == 5:               |
+------------------+   |    - status = DEAD                 |
                       |    - Move p/ webhook_dead_letter   |
                       | 4. Registra o delivery log         |
                       +------------------------------------+
```

### 4.3. Fluxo de Replay da Dead Letter Queue (DLQ)
1. Usuário autenticado com perfil `ADMIN` executa `POST /admin/webhooks/dead-letter/:id/replay`.
2. O sistema busca o registro correspondente na tabela `webhook_dead_letter`.
3. Remove o registro da tabela de DLQ.
4. Cria um novo registro na tabela `webhook_outbox` com status `PENDING`, `attempts = 0` e `nextAttemptAt = NOW()`.
5. Registra o log de auditoria Pino identificando o ID do usuário administrador que solicitou a execução.

---

## 5. Contratos Públicos (API)

### 5.1. Cadastrar Webhook
Cria uma nova assinatura de webhook para notificações de um cliente específico.
* **Endpoint**: `POST /webhooks`
* **Autenticação**: Requer JWT (qualquer perfil ativo).
* **Request Header**: `Content-Type: application/json`
* **Request Payload Exemplo**:
```json
{
  "customerId": "d3b07384-d113-4ec2-a5d5-bd8a329d5b0c",
  "url": "https://api.atlascomercial.com.br/webhooks/orders",
  "events": ["SHIPPED", "DELIVERED"]
}
```
* **Response Status**: `201 Created`
* **Response Payload Exemplo**:
```json
{
  "id": "fe56bc44-bc8c-42cb-b1b7-98e6dcb44222",
  "customerId": "d3b07384-d113-4ec2-a5d5-bd8a329d5b0c",
  "url": "https://api.atlascomercial.com.br/webhooks/orders",
  "events": ["SHIPPED", "DELIVERED"],
  "secret": "whsec_7d5e4a50d228f7fb74ce9e491ee4c7be5c0d1b3f9d8a5c2b0c345a27",
  "active": true,
  "createdAt": "2026-08-15T12:00:00.000Z"
}
```

### 5.2. Rotacionar Secret de Webhook
Gera uma nova secret para o webhook selecionado e ativa o grace period de 24 horas para a secret antiga.
* **Endpoint**: `POST /webhooks/:id/rotate`
* **Autenticação**: Requer JWT (qualquer perfil ativo).
* **Request Payload**: Nulo / Vazio.
* **Response Status**: `200 OK`
* **Response Payload Exemplo**:
```json
{
  "id": "fe56bc44-bc8c-42cb-b1b7-98e6dcb44222",
  "newSecret": "whsec_bc87d12f348e3a2c4e5b6d7a8f9c0b1a2d3e4f5a6b7c8d9e0f1a2b3c",
  "oldSecretValidUntil": "2026-08-16T12:00:00.000Z",
  "message": "A nova secret foi gerada. A secret anterior continuará válida para validações de assinatura por 24 horas (grace period)."
}
```

### 5.3. Consultar Histórico de Entregas (Deliveries)
Exibe a lista contendo as últimas 100 tentativas de envio realizadas para um webhook específico.
* **Endpoint**: `GET /webhooks/:id/deliveries`
* **Autenticação**: Requer JWT.
* **Response Status**: `200 OK`
* **Response Payload Exemplo**:
```json
{
  "webhookId": "fe56bc44-bc8c-42cb-b1b7-98e6dcb44222",
  "deliveries": [
    {
      "id": "78a59b2c-8cd4-4fa4-a4b5-bf6e4d2a1389",
      "eventId": "123e4567-e89b-12d3-a456-426614174000",
      "eventType": "order.status_changed",
      "attempt": 1,
      "success": true,
      "httpStatus": 200,
      "durationMs": 145,
      "createdAt": "2026-08-15T12:15:30.000Z"
    },
    {
      "id": "cfa34b8c-512c-47ea-baef-c3daef92348a",
      "eventId": "223e4567-e89b-12d3-a456-426614174999",
      "eventType": "order.status_changed",
      "attempt": 2,
      "success": false,
      "httpStatus": 504,
      "errorMessage": "Gateway Timeout",
      "durationMs": 10012,
      "createdAt": "2026-08-15T12:10:00.000Z"
    }
  ]
}
```

### 5.4. Reprocessar Evento da DLQ (Replay)
Disponibiliza o reprocessamento manual de uma notificação morta. Exclusivo para administradores do sistema.
* **Endpoint**: `POST /admin/webhooks/dead-letter/:id/replay`
* **Autenticação**: Requer JWT contendo a role `ADMIN`.
* **Request Payload**: Nulo / Vazio.
* **Response Status**: `200 OK`
* **Response Payload Exemplo**:
```json
{
  "success": true,
  "message": "Evento reprocessado com sucesso e recolocado na outbox de saída com status PENDING.",
  "replayInfo": {
    "deadLetterId": "8ba12c94-d2e3-4d7a-b5e2-e1cb329fa0c1",
    "newOutboxId": "d30fa128-b8bc-49ea-99dc-71f021c9fa00",
    "replayedAt": "2026-08-15T14:30:15.000Z",
    "requestedBy": "98a44bcf-a4d3-4fc2-a27b-e1bc119fcabc"
  }
}
```

### 5.5. Atualizar Webhook (PATCH)
Permite ao cliente alterar as configurações do webhook, como URL, eventos assinados ou estado ativo.
* **Endpoint**: `PATCH /webhooks/:id`
* **Autenticação**: Requer JWT (qualquer perfil ativo).
* **Request Header**: `Content-Type: application/json`
* **Request Payload Exemplo**:
```json
{
  "url": "https://api.atlascomercial.com.br/new-route/orders",
  "events": ["SHIPPED", "DELIVERED", "CANCELLED"],
  "active": true
}
```
* **Response Status**: `200 OK`
* **Response Payload Exemplo**:
```json
{
  "id": "fe56bc44-bc8c-42cb-b1b7-98e6dcb44222",
  "customerId": "d3b07384-d113-4ec2-a5d5-bd8a329d5b0c",
  "url": "https://api.atlascomercial.com.br/new-route/orders",
  "events": ["SHIPPED", "DELIVERED", "CANCELLED"],
  "active": true,
  "createdAt": "2026-08-15T12:00:00.000Z",
  "updatedAt": "2026-08-15T14:22:10.000Z"
}
```

### 5.6. Remover Webhook (DELETE)
Exclui permanentemente um cadastro de webhook do banco de dados.
* **Endpoint**: `DELETE /webhooks/:id`
* **Autenticação**: Requer JWT (qualquer perfil ativo).
* **Request Payload**: Nulo / Vazio.
* **Response Status**: `204 No Content`
* **Response Payload Exemplo**: Nulo / Vazio.

### 5.7. Listar Webhooks de um Customer (GET List)
Recupera todos os webhooks configurados associados a um determinado cliente (Customer).
* **Endpoint**: `GET /webhooks/customer/:customerId`
* **Autenticação**: Requer JWT.
* **Response Status**: `200 OK`
* **Response Payload Exemplo**:
```json
{
  "customerId": "d3b07384-d113-4ec2-a5d5-bd8a329d5b0c",
  "webhooks": [
    {
      "id": "fe56bc44-bc8c-42cb-b1b7-98e6dcb44222",
      "url": "https://api.atlascomercial.com.br/new-route/orders",
      "events": ["SHIPPED", "DELIVERED", "CANCELLED"],
      "active": true,
      "createdAt": "2026-08-15T12:00:00.000Z"
    }
  ]
}
```

---

## 6. Matriz de Erros Previstos
As respostas de erro do módulo de webhooks utilizarão códigos de erro estruturados com o prefixo `WEBHOOK_`, seguindo estritamente as convenções do projeto.

| Status Code | Código de Erro (Zod / AppError) | Causa Provável do Erro |
| --- | --- | --- |
| `400 Bad Request` | `WEBHOOK_INVALID_URL` | A URL fornecida não utiliza o protocolo obrigatório HTTPS/TLS. |
| `400 Bad Request` | `WEBHOOK_INVALID_EVENTS` | A lista de eventos assinados contém status inexistentes ou vazios. |
| `403 Forbidden` | `WEBHOOK_FORBIDDEN` | Usuário autenticado tenta gerenciar webhooks de outro customer sem permissão. |
| `404 Not Found` | `WEBHOOK_NOT_FOUND` | O identificador do webhook ou registro solicitado não existe no banco de dados. |
| `404 Not Found` | `WEBHOOK_CUSTOMER_NOT_FOUND` | O `customerId` fornecido no cadastro de webhook não foi localizado. |
| `409 Conflict` | `WEBHOOK_ALREADY_EXISTS` | Cadastro idêntico de URL de webhook para aquele mesmo cliente já ativo. |
| `413 Payload Too Large` | `WEBHOOK_PAYLOAD_TOO_LARGE` | O payload final gerado excede o limite físico estipulado de 64KB. |

---

## 7. Estratégias de Resiliência
- **Network Timeouts**: O disparo HTTP feito pelo Worker terá timeout inflexível configurado para **10 segundos**. Se o cliente não responder nesse período, a chamada é encerrada, registrada como falha e agendada para retry.
- **Backoff Exponencial**: Intervalos progressivos de agendamento entre retries: **1 minuto**, **5 minutos**, **30 minutos**, **2 horas** e **12 horas**, totalizando 5 tentativas distribuídas ao longo de 15 horas de margem.
- **Dead Letter Queue (DLQ)**: Isolamento permanente de mensagens com mais de 5 falhas definitivas na tabela `webhook_dead_letter` para não poluir ou diminuir a vazão do Worker principal.
- **Idempotência no Receptor**: Cabeçalho `X-Event-Id` contendo o ID exclusivo do evento, permitindo que os parceiros façam a deduplicação de mensagens duplicadas.

---

## 8. Observabilidade
- **Métricas**: Serão coletadas e expostas métricas chaves em um formato compatível com Prometheus para garantir visibilidade operacional do Worker de webhooks:
  - `webhook_events_dispatched_total`: Contador incremental acumulado por `eventType` e `status` (success, failure) para acompanhar a vazão geral de envios.
  - `webhook_delivery_duration_seconds`: Histograma/Percentis (p50, p90, p95, p99) para mensurar o tempo de resposta das requisições HTTP enviadas aos clientes.
  - `webhook_delivery_failures_total`: Contador agrupado por `webhookId`, `customerId` e `http_status` ou erros de rede (ex: `ETIMEDOUT`) para rastreamento de instabilidade das pontas.
  - `webhook_retries_attempted_total`: Contador agrupado por número da tentativa (1 a 5) para monitoramento do comportamento de instabilidade passageira.
  - `webhook_dead_letter_insertions_total`: Contador de eventos que falharam definitivamente e foram direcionados para a DLQ.
  - `webhook_active_outbox_queue_size`: Métrica do tipo Gauge indicando o total de registros pendentes na outbox aguardando processamento ou agendamento de retry.
- **Geração de Logs**: Uso da biblioteca central Pino. O Worker emitirá logs detalhados para cada fase:
  - Loop Start: `[Polling] Buscando lotes de eventos pendentes na outbox...`
  - Event Process: `[Worker] Enviando evento {eventId} para {url}. Tentativa {attempt}/5`
  - Event Success: `[Worker] Evento {eventId} enviado com sucesso em {durationMs}ms. Status HTTP {status}`
  - Event Error: `[Worker] Falha no envio de {eventId} para {url}. Erro: {error}. Próxima tentativa agendada para {nextAttemptAt}`
  - Event Dead: `[Worker] Evento {eventId} esgotou as tentativas e foi movido para a DLQ`
- **Tracing**: Inserção do header `X-Correlation-ID` unificado (se disponível na requisição original) para rastrear o fluxo completo desde o PATCH de alteração de status até o disparo final do webhook.

---

## 9. Integração com o Sistema Existente
Esta seção descreve os ganchos físicos de integração do módulo de webhooks com arquivos reais presentes na codebase do projeto.

1. **`prisma/schema.prisma`**:
   - Adicionaremos os modelos `Webhook`, `WebhookOutbox` (ou similar) e `WebhookDeadLetter` para persistência dos cadastros de endpoints, histórico de outbox assíncrono e eventos mortos, respectivamente.
   - Utilizaremos os mapeamentos nativos `@db.Char(36)` para id (UUID) e tipos do Prisma que casam com o banco MySQL atual.
2. **`src/modules/orders/order.service.ts`**:
   - Integração dentro do método transacional `changeStatus` (linhas 150-185). Logo após atualizar o status da order (`tx.order.update`) e escrever o histórico (`tx.orderStatusHistory.create`), inseriremos a chamada do serviço de outbox:
     `await this.webhookOutboxService.publishWebhookEvent(tx, refreshedOrder, from, to)`
   - Essa integração passa a instância de transação do Prisma `tx` (Prisma.TransactionClient) garantindo atomicidade estrita de banco de dados. Se a gravação no outbox lançar erros, a mudança do status do pedido sofre rollback completo.
3. **`src/shared/errors/index.ts` e `src/shared/errors/app-error.ts`**:
   - Reuso completo da estrutura de tratamento de erros global do projeto.
   - Herdaremos a classe de erros personalizada `AppError` para definir nossos erros de negócios específicos de webhooks (ex: `WebhookNotFoundError` ou `InvalidUrlError`), o que alimentará automaticamente o middleware central de tratamento de erros em `src/middlewares/error.middleware.ts`.
4. **`src/middlewares/auth.middleware.ts`**:
   - O endpoint administrativo de Replay (`POST /admin/webhooks/dead-letter/:id/replay`) utilizará os middlewares de autenticação existentes, especificamente o middleware `requireRole` com o tipo `UserRole.ADMIN` definido no Prisma e verificado por meio de tokens JWT.
5. **`src/routes/index.ts`**:
   - Modificaremos as rotas raiz do projeto para incorporar as rotas de webhooks através do método `router.use('/webhooks', webhookRoutes)` e `router.use('/admin/webhooks', adminWebhookRoutes)`.
6. **`src/shared/logger/index.ts`**:
   - Importaremos e utilizaremos a instância do logger Pino configurada no projeto em todo o ciclo de execução do `src/worker.ts` e classes do módulo, garantindo logs perfeitamente compatíveis e formatados.

---

## 10. Riscos e Mitigações
- **Risco: Inundação de Conexões Concorrentes (Connection Exhaustion)**: 
  - *Mitigação*: Limitaremos o tamanho máximo do pool de conexões do Prisma no Worker isolado e configuraremos agentes de conexão HTTP no Node com `keepAlive: true` e limite físico de concorrência.
- **Risco: Crescimento Excessivo da Tabela Outbox**:
  - *Mitigação*: O Worker remove ou marca os registros finalizados com status `DELIVERED`, de modo que a busca de polling só lê registros ativos, garantindo que o volume de leitura não degrade.
- **Risco: Concorrência de Execução Duplicada**:
  - *Mitigação*: Garantir que o Worker rodará de forma isolada (single-worker) ou utilizar locks pessimistas (`SELECT FOR UPDATE` com skip locked) nas consultas de polling do Prisma para impedir que múltiplos processos paralelos enviem o mesmo evento simultaneamente.

---

## 11. Dependências e Compatibilidade
Este módulo estende as capacidades do OMS utilizando as seguintes dependências físicas e restrições de compatibilidade:
- **Node.js (versão >= 18.x)**: Compatibilidade com ESM (ES Modules) e suporte a APIs nativas de criptografia (`crypto`) para cálculo de HMAC-SHA256 e geração de secrets seguras.
- **Prisma Client (versão >= 5.x)**: Reutilização do pool de conexões e da biblioteca Prisma nativa do projeto.
- **MySQL (versão >= 8.0)**: Compatibilidade com recursos avançados de concorrência, como índices funcionais em propriedades JSON e suporte ao comando `SELECT FOR UPDATE SKIP LOCKED` para impedir race conditions em processamentos do Worker.
- **Axios ou Undici**: Para realizar requisições HTTP de saída eficientes, permitindo controle de timeout configurado para 10s e agentes Keep-Alive reutilizáveis para reaproveitamento de sockets TCP.

---

## 12. Critérios de Aceite Técnicos
O sistema será considerado homologado tecnicamente se atender integralmente a todos os critérios abaixo:
- **Isolamento de Processo**: O Worker deve executar de forma totalmente desacoplada da API principal através do comando `npm run worker` (ponto de entrada `src/worker.ts`), sem carregar rotas do Express em memória.
- **Transacionalidade Estrita**: O evento de webhook correspondente deve ser inserido na tabela `webhook_outbox` na mesma transação atômica que a atualização de status do pedido. Se a alteração de status do pedido falhar por estoque, transição inválida ou qualquer outro erro, a outbox sofrerá rollback completo de banco de dados.
- **Validação de Protocolo Seguro**: O cadastro de webhooks deve recusar URLs que utilizem o protocolo inseguro HTTP, exigindo estritamente HTTPS em nível de schema de validação Zod.
- **Proteção Criptográfica**: Cada requisição de saída enviada pelo Worker deve conter uma assinatura hex válida gerada com HMAC-SHA256 alimentada pela secret do webhook, transmitida no header `X-Signature`.
- **Controle de Acesso Administrativo**: O endpoint administrativo de replay da DLQ (`POST /admin/webhooks/dead-letter/:id/replay`) deve bloquear perfis não autorizados (como `OPERATOR`) com retorno `403 Forbidden`, exigindo autenticação robusta e perfil `ADMIN` verificado via token JWT.

