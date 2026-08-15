# ADR-004: Garantia At-Least-Once com X-Event-Id

## Status
Aprovado

## Contexto
Durante o envio de webhooks por uma rede instável, podem ocorrer cenários de falhas parciais. Por exemplo:
1. O Worker envia a requisição de webhook para o endpoint do cliente.
2. O cliente recebe a requisição, processa-a com sucesso e muda o status local do pedido, mas seu servidor cai ou sofre lentidão na rede imediatamente antes de enviar a resposta HTTP `200 OK`.
3. Do ponto de vista do Worker, o envio resultou em falha de conexão ou timeout.
4. O Worker, seguindo a política de retries, agenda o envio do mesmo evento novamente.
5. O cliente receberá a mesma notificação duplicada.

Como não é viável garantir a entrega exata de uma única mensagem (Exactly-Once) sem coordenação transacional distribuída pesada (duas fases commit, etc), precisamos estabelecer as garantias de entrega do nosso sistema e fornecer mecanismos para que o cliente neutralize a duplicidade de dados sem corromper seus registros internos.

## Decisão
Garantiremos a entrega no nível **At-Least-Once** (pelo menos uma vez). Isso significa que nos comprometemos a garantir que o evento será entregue ao cliente com sucesso, mesmo que isso implique no reenvio do mesmo evento em caso de falha de confirmação (ack).

Para permitir que o cliente neutralize e filtre requisições duplicadas (idempotência), geraremos um identificador único universal (UUID) para cada evento no exato momento em que ele entra na outbox. Esse ID será enviado em um header HTTP chamado `X-Event-Id`.

Com esse header, o cliente é instruído formalmente a registrar os IDs de eventos processados com sucesso em sua base de dados temporária ou cache de deduplicação, rejeitando ou ignorando requisições que cheguem contendo um `X-Event-Id` que ele já processou anteriormente.

Adicionalmente, enviaremos o header `X-Timestamp` contendo a data e hora do processamento da tentativa de envio atual, permitindo que o cliente se previna contra ataques de replay baseados em pacotes de rede antigos.

## Alternativas Consideradas

### 1. Entrega Exatamente-Uma-Vez (Exactly-Once)
* **Prós**: Evita qualquer necessidade de tratamento de duplicidade ou deduplicação do lado do cliente.
* **Contras**: Impossível de garantir puramente sobre o protocolo HTTP de forma isolada, exigindo transações distribuídas bidirecionais que introduzem gargalos críticos de latência, complexidade extrema e dependência direta da disponibilidade em tempo real da infraestrutura do receptor.

## Consequências
* **Positivas**:
  * **Garantia de Entrega Resiliente**: O sistema de webhooks garante robustez extrema ao insistir no reenvio até obter uma resposta legítima do cliente (status 2xx).
  * **Idempotência de Baixo Custo**: O envio do `X-Event-Id` joga a responsabilidade de deduplicação para a ponta receptora de forma simples, leve e aderente aos principais padrões mundiais de API (como Stripe).
* **Negativas**:
  * **Trabalho Adicional do Cliente**: O cliente deve obrigatoriamente criar uma lógica de filtro de idempotência com base no header `X-Event-Id` se quiser evitar processamentos duplicados das notificações enviadas em caso de retries.
