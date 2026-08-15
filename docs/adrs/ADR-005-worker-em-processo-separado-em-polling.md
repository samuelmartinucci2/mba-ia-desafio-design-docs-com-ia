# ADR-005: Worker em Processo Separado em Polling

## Status
Aprovado

## Contexto
A leitura e envio de webhooks é uma operação intensiva em I/O (I/O Bound) devido às chamadas HTTP de rede que podem apresentar timeouts longos (configurados para até 10 segundos). 
Se executássemos o loop de leitura e processamento de eventos do webhook dentro do mesmo processo Node.js que atende às requisições HTTP normais da nossa API (`src/server.ts`):
1. Estaríamos compartilhando o Event Loop de requisições de clientes com o loop de reenvio pesado de webhooks, criando riscos de gargalo em cenários de alta carga.
2. Qualquer instabilidade severa ou vazamento de memória gerado pelo processamento assíncrono de rede dos webhooks poderia derrubar a API principal inteira, inviabilizando o OMS e impedindo novos pedidos de serem inseridos.
3. Se a instância da API principal precisasse reiniciar ou escalar horizontalmente de forma rápida, estaríamos escalando desnecessariamente threads de processamento de fila junto com endpoints HTTP de leitura.

Além disso, precisamos definir como o Worker identificará novos registros de eventos inseridos na outbox do MySQL de forma rápida, eficiente e integrada ao Prisma.

## Decisão
Implementar o Worker como um **processo Node.js totalmente independente** (Background Worker) em uma nova entrada do projeto: `src/worker.ts`, operando através de um loop contínuo de **Polling** ativo com intervalo regulado de **2 segundos**.

O Worker poderá ser iniciado via linha de comando através de um novo script exclusivo adicionado ao `package.json`: `npm run worker`.

Ele compartilhará a mesma stack de persistência e banco de dados (MySQL) através de uma nova instância isolada do `PrismaClient` (já que o PrismaClient é gerido por processo Node). 
A cada 2 segundos, o Worker executará uma consulta buscando em lote (batch) os registros que atendam aos critérios de envio (status `PENDING` ou `FAILED` com agendamento de retry no passado/presente), executará os disparos HTTP paralelos (usando limites controlados de concorrência) e atualizará seus respectivos status na tabela.

O uso de polling simples de 2 segundos cumpre com folga o requisito de negócio acordado com os clientes de entregar notificações com latência inferior a 10 segundos.

## Alternativas Consideradas

### 1. Trigger de Banco de Dados ou Event-Driven Dinâmico
* **Prós**: Notificação instantânea (sub-segundo) ao Worker sem necessidade de loops periódicos que geram consultas vazias no banco de dados.
* **Contras**: O MySQL não possui suporte nativo confiável para notificação push para processos externos fora do banco (como o `LISTEN/NOTIFY` do PostgreSQL). Adaptar triggers no MySQL exigiria gambiarras complexas como rodar rotinas externas que leem logs binários (CDC) ou interagir diretamente com arquivos do sistema operacional, introduzindo fragilidade operacional severa.

### 2. Rodar Worker em thread paralela interna (Worker Threads da API)
* **Prós**: Dispensa a necessidade de rodar processos separados e configurar scripts adicionais de inicialização ou infraestrutura de deploy múltipla.
* **Contras**: Mantém o acoplamento físico do ciclo de vida da API com o do Worker. Se a API sofrer reinicializações, os processamentos e agendamentos de retries do webhook são abruptamente cortados, e o isolamento de concorrência de CPU/Event Loop fica vulnerável.

## Consequências
* **Positivas**:
  * **Isolamento Completo**: A API principal do OMS continua performática e isolada de qualquer falha no envio de webhooks ou lentidão na rede dos clientes.
  * **Escalabilidade Independente**: Podemos gerenciar a infraestrutura de deploy (ex: containers Docker) de forma distinta, subindo múltiplas instâncias da API de acordo com requisições HTTP, mantendo o Worker rodando de forma única (single-worker) para assegurar o ordenamento cronológico implícito de forma simples no banco.
  * **Implementação Simples**: O polling de 2 segundos é fácil de debugar, testar e implementar usando consultas padrão do Prisma sem introduzir dependências externas complicadas.
* **Negativas**:
  * **Consultas Vazias**: Em períodos em que não houver modificações de status de pedidos, o Worker continuará batendo no MySQL a cada 2 segundos, gerando atividade residual (mitigada por índices e tabelas otimizadas).
  * **Acoplamento de Recursos de Banco**: Embora os processos Node estejam separados, o Worker compartilha do mesmo banco de dados MySQL que a API, exigindo dimensionamento adequado do pool de conexões do Prisma.
