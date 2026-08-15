# ADR-002: Política de Retry com Backoff e DLQ

## Status
Aprovado

## Contexto
Servidores de clientes externos que recebem webhooks estão sujeitos a quedas, instabilidades temporárias de rede, timeouts ou manutenções programadas. Para garantir que as notificações cheguem mesmo após problemas temporários, precisamos de uma política de retentativa resiliente. 

No entanto, tentativas infinitas ou muito frequentes podem sobrecarregar nossos próprios recursos (trancando linhas do banco de dados na tabela de outbox) ou bombardear desnecessariamente o servidor do cliente que já está instável. Por outro lado, tentativas insuficientes (como apenas 3 vezes em um curto espaço de tempo) não cobririam janelas de manutenção rotineiras ou indisponibilidades prolongadas (ex: 12 horas).

## Decisão
Implementar uma política de **5 tentativas de reenvio**, utilizando a estratégia de **Backoff Exponencial**, com intervalos definidos e crescentes. Caso todas as 5 tentativas falhem, o evento será considerado permanentemente com falha e movido para uma fila de mensagens mortas (**Dead Letter Queue - DLQ**), persistida na tabela `webhook_dead_letter`.

As janelas de reenvio propostas são:
- **Tentativa 1 (falha inicial)**: reenvio após 1 minuto.
- **Tentativa 2**: reenvio após 5 minutos.
- **Tentativa 3**: reenvio após 30 minutos.
- **Tentativa 4**: reenvio após 2 horas.
- **Tentativa 5**: reenvio após 12 horas.

A janela de cobertura total do retry será de aproximadamente **15 horas** (14 horas e 36 minutos no acumulado). 

A tabela `webhook_dead_letter` guardará:
- `id` (UUID)
- `outboxId` (UUID)
- `webhookId` (UUID)
- `customerId` (UUID)
- `payload` (JSON)
- `lastError` (Text - contendo código de status HTTP ou mensagem de timeout/erro de rede)
- `failedAt` (DateTime)

Para mitigar a inatividade permanente de webhooks na DLQ, disponibilizaremos um endpoint administrativo exclusivo para usuários com perfil `ADMIN` (`POST /admin/webhooks/dead-letter/:id/replay`) que recoloca o evento na tabela outbox principal com status `PENDING` e contador de tentativas resetado para zero, permitindo o reprocessamento manual auditado.

## Alternativas Consideradas

### 1. Retentativas Infinitas
* **Prós**: Garante que o cliente receberá a mensagem mesmo se ficar offline por dias ou semanas.
* **Contras**: Eventos obsoletos travam o processamento do Worker, congestionam a outbox ativa com registros "vampiros" e geram loopings desnecessários para endpoints de clientes desativados.

### 2. Retry Agressivo / Poucas Tentativas (ex: 3 tentativas em 30 minutos)
* **Prós**: Libera espaço e processamento rapidamente, limpando a tabela de outbox.
* **Contras**: Períodos comuns de indisponibilidade ou deploy de clientes que durem mais de 1 hora levariam à perda silenciosa de eventos cruciais de mudança de status de pedido, aumentando o volume de chamadas de suporte ou reprocessamentos manuais.

## Consequências
* **Positivas**:
  * **Ampla Janela de Cobertura**: Cobre janelas de até 15 horas de indisponibilidade do cliente, protegendo contra quedas e manutenções ao longo de noites ou fins de semana parciais.
  * **Isolamento de Mensagens Mortas**: Evita que o Worker fique tentando reenviar indefinidamente dados para endpoints quebrados, mantendo a tabela outbox principal limpa e performática.
  * **Trilha de Auditoria e Replay**: Permite que administradores investiguem falhas de entrega de webhook na DLQ e as executem novamente (replay) de forma manual após o cliente restabelecer seu serviço.
* **Negativas**:
  * **Delay para DLQ**: Uma mensagem que falhe definitivamente levará no mínimo 15 horas para cair na tabela de DLQ, atrasando a intervenção manual do time de suporte, a menos que monitorada previamente.
  * **Complexidade no Worker**: O Worker de processamento precisará calcular timestamps dinâmicos (`nextAttemptAt`) baseados na tentativa atual para buscar apenas os eventos elegíveis de outbox.
