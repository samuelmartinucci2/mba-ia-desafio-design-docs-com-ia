# ADR-001: Padrão Outbox no MySQL

## Status
Aprovado

## Contexto
Durante a discussão técnica para a nova funcionalidade de Webhooks de Notificação de Pedidos, identificou-se a necessidade de disparar notificações de mudança de status de pedidos de forma confiável. 
Se disparássemos as chamadas HTTP de forma síncrona diretamente no fluxo de atualização do pedido em `src/modules/orders/order.service.ts`:
1. Bloquearíamos e sobrecarregaríamos a transação SQL atual que já realiza tarefas pesadas (como debitar estoque, registrar histórico e atualizar pedido).
2. Qualquer lentidão ou instabilidade na URL do cliente afetaria diretamente a experiência de atualização e finalização dos pedidos dos clientes.
3. Se a chamada HTTP falhasse, não poderíamos reverter a transação de banco com segurança, gerando inconsistências severas de dados (pedido atualizado mas notificação perdida, ou rollback indesejado de pedidos válidos devido a falhas do cliente).

Por isso, é necessário um mecanismo assíncrono e transacional que garanta que, se a transação do pedido persistiu com sucesso, a intenção de envio do webhook correspondente também seja registrada de forma atômica.

## Decisão
Adotaremos o padrão **Transactional Outbox** integrado diretamente ao banco MySQL do projeto via Prisma. 

Ao mudar o status do pedido dentro da transação em `src/modules/orders/order.service.ts`, adicionaremos uma nova linha contendo o evento de notificação em uma tabela do banco chamada `webhook_outbox`. 
Essa tabela terá o id no formato UUID (`db.Char(36)`), seguindo o padrão estabelecido do projeto, e guardará a payload já renderizada (snapshot no momento da inserção). Isso garante a integridade histórica do evento mesmo que o pedido sofra atualizações posteriores antes de o evento ser enviado.

A estrutura do evento armazenado conterá:
- `id` (UUID)
- `webhookId` (UUID)
- `customerId` (UUID)
- `eventType` (`order.status_changed`)
- `payload` (JSON)
- `status` (PENDING, PROCESSING, DELIVERED, FAILED)
- `attempts` (Int)
- `createdAt` (DateTime)
- `updatedAt` (DateTime)

## Alternativas Consideradas

### 1. Chamada Síncrona Direta na API
* **Prós**: Implementação extremamente simples e direta sem necessidade de novas tabelas ou workers.
* **Contras**: Acoplamento temporário forte, lentidão nas transações, risco de timeout da transação principal, sem capacidade de retry resiliente ou isolamento contra falhas de terceiros.

### 2. Mensageria Dedicada (Redis Streams / RabbitMQ)
* **Prós**: Alta performance, desacoplamento completo e processamento nativo de eventos assíncronos.
* **Contras**: Exige o provisionamento e manutenção de infraestrutura adicional (como um cluster Redis ou RabbitMQ). Sendo o time pequeno, isso introduz custo operacional de infraestrutura desnecessário (overengineering) para o volume inicial de clientes B2B planejado.

## Consequências
* **Positivas**:
  * **Confiabilidade Atômica**: O evento de webhook é inserido na mesma transação que a atualização do status do pedido. Se a transação der commit, o evento será enviado eventualmente; se der rollback, o evento não é inserido.
  * **Isolamento de Erros**: Falhas na rede ou servidores dos clientes não impactam a criação ou transição de pedidos na nossa API.
  * **Resiliência**: Permite a implementação de retries assíncronos a partir da tabela física.
* **Negativas**:
  * **Latência de Envio**: O processamento deixa de ser instantâneo e passa a depender do loop do Worker assíncrono.
  * **Sobrecarga no Banco**: Introduz escrita adicional de transação e leitura concorrente no MySQL existente, embora em escala mitigada pelos índices estruturados em `status` e `createdAt`.
