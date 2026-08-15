# ADR-003: Autenticação HMAC-SHA256 com Secret por Endpoint

## Status
Aprovado

## Contexto
O tráfego de requisições enviadas por webhooks sai de nossa infraestrutura privada e trafega pela internet pública em direção aos servidores dos clientes. Isso introduz sérios riscos de segurança:
1. Um agente mal-intencionado pode tentar forjar payloads falsificando mudanças de status (ex: marcar um pedido como pago sem que de fato tenha sido) e enviá-los ao servidor do cliente simulando ser o nosso sistema.
2. Os dados de pedidos, mesmo trafegando por HTTPS, precisam de garantias extras de integridade para provar ao receptor que o conteúdo do payload não foi adulterado no meio do caminho.

Para resolver isso, os clientes precisam de uma forma criptograficamente segura de validar que a chamada recebida partiu legitimamente do nosso sistema e que o corpo da mensagem está idêntico ao emitido.

## Decisão
Implementar a autenticação de saída utilizando assinaturas digitais com **HMAC-SHA256** calculadas sobre o corpo bruto (`JSON.stringify`) do request HTTP. 

Cada cadastro de webhook configurado pelo cliente possuirá uma **chave secreta única e gerada aleatoriamente** (`secret`), que nunca será compartilhada com outros clientes ou exposta de forma global. 

Durante o envio da notificação, o Worker irá assinar o payload usando o algoritmo HMAC-SHA256 alimentado pela secret do webhook. O hash hexadecimal resultante será enviado no cabeçalho HTTP da requisição sob o header `X-Signature`.

Além disso, introduziremos um mecanismo seguro de **rotação de secrets** através de API. Quando um cliente rotacionar sua chave secreta, manteremos a secret antiga válida por um período de carência (grace period) de **24 horas em paralelo**, permitindo que o cliente migre suas configurações sem sofrer indisponibilidades. Após 24 horas, a secret antiga é permanentemente invalidada.

## Alternativas Consideradas

### 1. Autenticação Básica (Basic Auth) ou Bearer Tokens estáticos
* **Prós**: Implementação muito simples e rápida de configurar por ambas as partes.
* **Contras**: Vulnerável caso o token trafegue por proxy ou vaze em logs do cliente, pois permite que atacantes clonem e forjem requisições livremente de forma retroativa. Não garante integridade do payload (um payload adulterado contendo um token estático ainda seria aceito).

### 2. Secret Global de Plataforma
* **Prós**: Armazenamento simples em variáveis de ambiente, sem necessidade de chaves únicas por cadastro no banco de dados.
* **Contras**: Se a chave de um único cliente for vazada ou comprometida por falha dele, toda a rede de webhooks de todos os clientes torna-se imediatamente vulnerável e exposta, exigindo intervenção drástica que afetaria todos simultaneamente.

## Consequências
* **Positivas**:
  * **Segurança Robusta**: Garante a autenticidade da origem e a integridade matemática de cada payload enviado, seguindo os melhores padrões de mercado adotados por empresas como Stripe e GitHub.
  * **Mitigação de Vazamento**: Se a chave de um cliente for comprometida, o dano é restrito apenas ao escopo daquele endpoint específico, e ele pode rotacionar a chave de forma autônoma pela API.
  * **Zero downtime na Rotação**: O período de carência de 24 horas para rotação de secrets impede interrupções indesejadas de processamento nos sistemas dos clientes durante manutenções operacionais.
* **Negativas**:
  * **Esforço de Implementação do Cliente**: O cliente precisará codificar ou utilizar uma biblioteca criptográfica para calcular e validar as assinaturas `X-Signature` de cada requisição.
  * **Armazenamento de Chaves de Criptografia**: As secrets precisarão ser gravadas no banco de dados. Devem ser tratadas com o devido sigilo de acesso e criptografadas em repouso no banco se necessário (futuro).
