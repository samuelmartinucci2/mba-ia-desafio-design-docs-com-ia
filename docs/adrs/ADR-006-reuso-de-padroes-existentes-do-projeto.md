# ADR-006: Reuso de Padrões Existentes do Projeto

## Status
Aprovado

## Contexto
O projeto atual possui um design de arquitetura muito bem estabelecido, limpo e estruturado. Cada domínio da aplicação está encapsulado de forma coesa como um módulo dentro do diretório `src/modules/` (ex: `src/modules/orders/`, `src/modules/products/`, etc.). 

Além disso, existem convenções rígidas estabelecidas para:
1. **Lógica de Erros**: Classes de exceção que herdam de uma classe base centralizada `AppError` em `src/shared/errors/app-error.ts`, contendo propriedades como `statusCode` e `errorCode` que alimentam um middleware de erro global (`src/middlewares/error.middleware.ts`).
2. **Log de Aplicação**: Uso estruturado da biblioteca Pino configurada em `src/shared/logger/index.ts`.
3. **Validação de Payload**: Schemas Zod contidos nos subdiretórios de módulo, integrados a um middleware central de validação (`src/middlewares/validate.middleware.ts`).
4. **Controle de Acesso por Roles**: Verificação de permissões do usuário operador usando o middleware de autorização `requireRole` com o enum `UserRole` (ADMIN, OPERATOR) mapeado no Prisma.

Subir uma funcionalidade transversal robusta de webhooks introduzindo novos padrões ou bibliotecas redundantes aumentaria a dívida técnica, dificultaria a manutenção por engenheiros que já conhecem a base de código e criaria inconsistências no comportamento das APIs.

## Decisão
Adotaremos e estenderemos rigorosamente todos os padrões, convenções e classes existentes no projeto atual para a implementação da feature de Webhooks de Notificação de Pedidos.

Isso significa que:
1. **Módulo Isolado**: Todo o código de rotas, controllers, services, repositories e schemas de webhook será contido na nova pasta modular `src/modules/webhooks/`, perfeitamente alinhada com os módulos de orders e products.
2. **Erros Padronizados**: Criaremos e utilizaremos erros que estendem de `AppError` com o prefixo unificado `WEBHOOK_` nos códigos de erro expostos (ex: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`), aproveitando o middleware global de tratamento de erros existente sem necessidade de alterá-lo.
3. **Observabilidade Pino**: Utilizaremos a instância de Logger centralizada do Pino para todas as ações do Worker e rotas de webhooks, garantindo logs legíveis e padronizados para monitoração em produção.
4. **Validações Zod**: Toda validação de payloads de entrada nas rotas CRUD (como cadastrar e rotacionar secrets) será regida por schemas Zod integrados com o middleware de validação existente.
5. **Controle de Roles**: O endpoint administrativo de replay da DLQ (`POST /admin/webhooks/dead-letter/:id/replay`) utilizará o middleware `requireRole(UserRole.ADMIN)` do arquivo `src/middlewares/auth.middleware.ts`.
6. **Pool do Prisma**: O Worker isolado utilizará uma conexão independente do PrismaClient, configurada com as variáveis de ambiente existentes, respeitando os esquemas definidos em `prisma/schema.prisma`.

## Alternativas Consideradas

### 1. Criar novo framework / infra de Erros e Logs para o Worker
* **Prós**: Maior liberdade técnica para estruturar o fluxo de execução assíncrono de maneira ultra-otimizada.
* **Contras**: Introduz desvios significativos do padrão corporativo estabelecido, gerando logs com formatos inconsistentes, dificultando a coleta centralizada por ferramentas externas e exigindo novos tratamentos em middleware.

## Consequências
* **Positivas**:
  * **Consistência Técnica**: Manutenção facilitada, permitindo que qualquer engenheiro do time compreenda e ajuste o código rapidamente sem precisar aprender novas abstrações.
  * **Aproveitamento de Esforço**: Zero necessidade de reescrever lógica de tratamento de erro HTTP, serialização de JSON, autorização de rotas ou logging estruturado.
  * **Segurança Homogênea**: Reaproveita o middleware testado e homologado de autenticação/autorização, diminuindo drasticamente riscos de brechas de controle de acesso.
* **Negativas**:
  * **Acoplamento de Stack**: Se a arquitetura geral sofrer grandes alterações ou quebras futuras, o módulo de webhooks e o worker serão afetados em cascata devido ao forte acoplamento com o núcleo compartilhado em `src/shared/`.
