# **🌐 Amazon API Gateway**

## **1. Conceito e Objetivo Principal**

O Amazon API Gateway é um serviço *serverless* que permite criar, publicar, manter, monitorar e proteger APIs REST, HTTP e WebSocket em escala. Ele atua como uma **porta de entrada** para acessar recursos de *backend* (como AWS Lambda ou HTTP endpoints) e oferece uma rica camada de recursos adicionais.

### **1.1 O que é o API Gateway?**

É um serviço *serverless* que expõe APIs públicas (HTTP endpoints) para clientes. Ele **atua como um *proxy***, recebendo solicitações do cliente e as encaminhando para os serviços de *backend*.

### **1.2 Por que usar o API Gateway?**

Ele oferece muito mais do que um simples *endpoint* HTTP (como um Application Load Balancer). Seus recursos incluem:

* **Segurança:** Autenticação, autorização e chaves de API.  
* **Gerenciamento:** Versionamento de API e Múltiplos Ambientes (*Stages*).  
* **Otimização:** *Caching* de respostas e *throttling* (*rate limiting*).  
* **Validação:** Transformação e validação de requisições e respostas.  
* **Geração:** Exportação da API e geração de SDKs de cliente.

### **1.3 Integrações de Backend (Integration Types)**

O API Gateway pode se integrar a três tipos principais de *backends*:

| Backend | Descrição | Exemplo de Uso |
| :---- | :---- | :---- |
| **AWS Lambda** | Invoca funções Lambda para construir aplicações *serverless* completas. É a integração mais comum. | *Backend* lógico para uma API REST. |
| **HTTP Endpoints** | Expõe qualquer *endpoint* HTTP (em nuvem ou *on-premises*) através do API Gateway. | Usado para adicionar recursos do API Gateway (caching, throttling, auth) sobre um *load balancer* ou servidor existente. |
| **AWS Service** | Expõe APIs de serviços AWS diretamente. | Enviar dados diretamente para **Kinesis Data Streams** ou iniciar um *workflow* do **Step Functions** sem expor credenciais AWS ao cliente. |

---

## **2\. Tipos de Endpoint (Deployment)**

O API Gateway pode ser implantado com diferentes tipos de *endpoints*, afetando a latência e a acessibilidade:

| Tipo de Endpoint | Acessibilidade | Otimização de Latência | Uso Típico |
| :---- | :---- | :---- | :---- |
| **Edge-Optimized** | Global (*Default*) | Utiliza a rede global do **CloudFront** (Edge Locations) para rotear solicitações e reduzir a latência para clientes globais. | Clientes globais. |
| **Regional** | Específico da Região | Não usa o CloudFront. Útil quando todos os clientes estão na mesma região da API. Permite que você configure sua própria distribuição CloudFront. | Clientes regionais. |
| **Private** | Apenas dentro da VPC | A API só pode ser acessada de dentro da sua VPC através de **Interface VPC Endpoints**. O acesso é definido por **Resource Policies**. | Aplicações internas ou serviços que não devem ser expostos à internet. |

---

## **3\. Gerenciamento e Versionamento de APIs (Stages)**

### **Stages (Estágios)**

* Mudanças feitas na configuração da API **não entram em vigor** até que um **Deployment** seja feito.  
* O *Deployment* é feito em um **Stage** (Estágio). Cada Stage é uma instância da API (ex: dev, test, prod, v1, v2).  
* Stages têm seus próprios URLs e configurações. Eles podem coexistir e permitem a **migração de clientes** sem quebrar a compatibilidade (ex: migrar clientes de /v1 para /v2).  
* O histórico de *deployments* é mantido, permitindo **rollbacks** contínuos.

### **Stage Variables (Variáveis de Estágio)**

* Semelhantes a variáveis de ambiente, permitem **alterar valores de configuração sem a necessidade de um novo *deployment***.  
* **Uso Típico:**  
  * Apontar para diferentes *backends* HTTP (ex: http://dev-service.com ou http://prod-service.com).  
  * Apontar para **Aliases Lambda** correspondentes ao *stage* (ex: o prod stage invoca o prod alias da Lambda).  
  * Os valores são acessados no formato $stageVariables.variableName.

### **Canary Deployments**

* É uma técnica de *deployment* *Blue/Green* aplicada ao API Gateway.  
* Você cria um **"Canary Stage"** (Estágio Canário) que recebe uma **pequena porcentagem do tráfego de produção** (ex: 5%) para testar a nova versão da API ou do *backend* (ex: Lambda v2).  
* Se não houver problemas, o tráfego é totalmente movido para o Stage Canário, e a versão antiga é desativada.  
* Permite monitoramento e *debugging* separados para a nova versão.

---

## **4\. Integração e Transformação de Dados**

### **Integration Types: MOCK, Proxy e Custom**

1. **MOCK:** Retorna uma resposta **sem sequer enviar uma requisição** ao *backend*. Útil apenas para desenvolvimento/testes iniciais.  
2. **AWS Proxy / HTTP Proxy (Lambda Proxy):**  
   * É o **padrão mais simples e comum** para Lambda.  
   * A requisição do cliente é **passada diretamente** como *input* (event) para o *backend* (Lambda ou HTTP endpoint) **sem modificações**.  
   * O *backend* é totalmente responsável pela lógica de requisição e resposta.  
   * **Não permite o uso de Mapping Templates**.  
3. **Custom Integration (Não-Proxy \- AWS/HTTP):**  
   * Permite o uso de **Mapping Templates** para transformar o corpo, os *headers* e os parâmetros da requisição antes de enviar ao *backend* e transformar a resposta antes de enviar ao cliente.

### **Mapping Templates (Transformação de Dados)**

* São usados nas integrações Custom para modificar as requisições e respostas.  
* São escritos em **VTL (Velocity Template Language)**, uma linguagem de *scripting* que permite lógica condicional (if) e *loops*.  
* **Caso de Uso de Exame:**  
  * **Integração JSON $\\leftrightarrow$ XML (SOAP):** O API Gateway pode receber uma requisição JSON de um cliente REST e usar um *Mapping Template* para **transformar o *payload* em XML** para se comunicar com um *backend* SOAP.  
  * **Mapeamento de Parâmetros:** Renomear ou reordenar *query string parameters* ou parâmetros de corpo para atender aos requisitos do *backend*.

---

## **5\. Validação de Requisições (Request Validation)**

O API Gateway pode validar as requisições antes de enviá-las ao *backend*, **reduzindo chamadas desnecessárias** e garantindo que o *backend* receba o formato de dados esperado.

* **Mecanismo:** Você define modelos **JSON Schema** (Modelos) usando a especificação **OpenAPI**.  
* **O que é Validado:** Parâmetros (*query strings*, *headers*, URI) e a conformidade do *payload* (corpo) com o modelo JSON Schema especificado.  
* **Resposta Imediata:** Se a requisição não for válida, o API Gateway retorna um erro **400 Bad Request** diretamente ao cliente.

### **OpenAPI Specification (Swagger)**

* É o padrão utilizado para **definir e documentar** APIs REST.  
* O API Gateway possui **integração nativa** com o OpenAPI:  
  * Você pode **importar** uma definição OpenAPI (YAML ou JSON) para criar uma API.  
  * Você pode **exportar** uma API existente como uma especificação OpenAPI, que pode ser usada para gerar SDKs de cliente.

---

## **6\. Caching (Cache) no API Gateway**

O *caching* é usado para **reduzir o número de chamadas ao *backend***, diminuindo a carga e melhorando a latência para respostas repetidas.

### **6.1 Configuração e Características**

* **Nível de Configuração:** O cache é habilitado no nível do **Stage** (Estágio) e é um recurso **pago** (normalmente usado em ambientes de **produção**).  
* **Capacidade:** O tamanho do cache varia de 0,5 GB até 237 GB.  
* **Criptografia:** O cache é **criptografável** (*at rest*).  
* **TTL (Time-To-Live):** Define por quanto tempo os resultados ficam no cache. O padrão é **300 segundos (5 minutos)**, com mínimo de zero (sem cache) e máximo de uma hora.  
* **Sobrescrita (*Override*):** As configurações de cache do *stage* podem ser **sobrescritas no nível do Método** (recurso/verbo HTTP).

### **6.2 Invalidação de Cache**

* **Manual:** Pode ser feita imediatamente através do console.  
* **Via Cliente:** Clientes autorizados podem invalidar o cache enviando o *header*: Cache-Control: max-age=0.  
* **Segurança:** Para evitar que qualquer cliente invalide o cache, é altamente recomendável **exigir autorização IAM** (InvalidateCache policy) para permitir que apenas clientes autorizados usem o *header* Cache-Control: max-age=0.

---

## **7\. Planos de Uso (Usage Plans) e Chaves de API (API Keys)**

Planos de Uso e Chaves de API são usados para **monetizar, limitar e controlar o acesso** de clientes à sua API.

### **7.1 Componentes e Fluxo**

* **API Key (Chave de API):** Uma *string* que identifica o cliente. Deve ser fornecida no *header* x-api-key em cada requisição. É usada para autenticar e medir o uso.  
* **Usage Plan (Plano de Uso):** Define os limites de acesso (throttling e quotas) para um ou mais *stages* e métodos.  
  * **Throttling:** Controla a **velocidade** de acesso (ex: 100 requisições/segundo). Aplicado no nível da API Key.  
  * **Quotas:** Controla o **volume total** de requisições em um período (ex: 10.000 requisições por mês).

### **7.2 Ordem de Configuração**

1. Crie a API, configure os métodos que **exigem API Key** e implante no *Stage*.  
2. Gere as **API Keys** e distribua aos clientes.  
3. Crie o **Usage Plan** (com limites de *throttling* e *quota*).  
4. **Associe** as **API Keys** e os **API Stages** ao **Usage Plan**. (Este é o passo crucial que liga tudo).

---

## **8\. Monitoramento e *Troubleshooting***

### **8.1 Logs e Tracing**

* **CloudWatch Logs:** Habilitado no nível do **Stage**. Permite registrar informações detalhadas de requisição e resposta (corpo, cabeçalhos, etc.). **Cuidado com dados sensíveis** ao habilitar logs detalhados (nível DEBUG).  
* **AWS X-Ray:** Habilitado para obter o **rastreamento (*tracing*)** completo das requisições, idealmente desde o API Gateway até o *backend* (ex: AWS Lambda).

### **8.2 Métricas do CloudWatch (Por Stage)**

| Métrica | O que mede | Observações |
| :---- | :---- | :---- |
| **CacheHitCount / CacheMissCount** | Eficiência do cache. | Hit alto \= cache eficiente. |
| **IntegrationLatency** | Tempo que o API Gateway leva para **enviar a requisição ao *backend*** e **receber a resposta de volta**. | Mede o tempo de resposta do *backend*. |
| **Latency** | Tempo total entre o **recebimento da requisição do cliente** e o **retorno da resposta ao cliente**. | Inclui IntegrationLatency \+ tempo gasto no API Gateway (auth, cache, mapping). |
| **Erros 4XX** | Erros do **Cliente** (ex: 400 Bad Request, 403 Access Denied, 429 Too Many Requests). | 429 Too Many Requests é o código retornado em caso de *throttling*. |
| **Erros 5XX** | Erros do **Servidor** (*Backend*) (ex: 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout). | O API Gateway tem um **timeout máximo de 29 segundos**. Se o *backend* não responder nesse período, ele retorna um 504 Integration Failure. |

### **8.3 Throttling (Limitação)**

* **Limites Padrão da Conta:** O API Gateway tem um limite de *throttling* padrão de **10.000 requisições/segundo** por conta (soft limit).  
* **Risco de Contaminação:** Se uma API estiver sobrecarregada e não tiver limites de *stage* ou de *usage plan* definidos, ela pode consumir todo o limite da conta e **causar *throttling* em outras APIs**. (Semelhante à concorrência reservada do Lambda).  
* **Cliente Throttling:** O cliente recebe o erro **429 Too Many Requests** e deve usar **Exponential Backoff** para tentar novamente.

---

## **9\. Segurança e Autorização**

O API Gateway oferece três principais mecanismos de autorização:

### **9.1 IAM Permissions (Permissões IAM)**

* **Uso:** Ideal para **aplicações internas** ou acessos *cross-account*, onde o cliente já está na AWS (ex: EC2, Lambda, usuários IAM).  
* **Autenticação:** O cliente usa suas credenciais IAM para assinar a requisição (usando **Signature V4 \- SigV4**), que são passadas nos *headers*.  
* **Autorização:** Controlada pelas **IAM Policies** anexadas ao usuário/função.  
* **Resource Policies:** Usadas em conjunto com IAM para:  
  * Permitir **acesso *Cross-Account*** (Contas diferentes).  
  * Filtrar por **IPs específicos** ou **VPC Endpoints** (para APIs privadas).

### **9.2 Cognito User Pools (Pools de Usuários Cognito)**

* **Uso:** Para **aplicações externas** (web/mobile) onde o Cognito gerencia o ciclo de vida do usuário (autenticação, tokens).  
* **Autenticação:** O cliente se autentica com o Cognito e recebe um *token* de conexão.  
* **Autorização:** O API Gateway **verifica diretamente o *token* do Cognito** e, se for válido, permite o acesso. Não requer código de autorização customizado.

### **9.3 Lambda Authorizer (Antigo Custom Authorizer)**

* **Uso:** O mais flexível, usado principalmente com **sistemas de autenticação de terceiros** (ex: Auth0, tokens JWT/Oauth customizados).  
* **Mecanismo:**  
  1. O cliente envia um *token* (via *header* ou *query string*) para o API Gateway.  
  2. O API Gateway invoca uma **função Lambda** (**Lambda Authorizer**).  
  3. A função Lambda (que você deve programar) **valida o *token*** (ex: conversando com o terceiro) e retorna uma **IAM Policy** para o cliente.  
  4. A IAM Policy é **armazenada em cache** (importante para reduzir custos e latência de invocação do Lambda).

---

## **10\. CORS (Cross-Origin Resource Sharing)**

* **Necessidade:** O CORS deve ser habilitado no API Gateway quando uma aplicação (ex: um *site* estático no S3) hospedada em um **domínio** (ex: www.example.com) tenta fazer uma requisição de API para um **domínio diferente** (ex: api.example.com).  
* **Mecanismo:** O *browser* envia uma requisição **OPTIONS** (*pre-flight*) antes da requisição real. O API Gateway responde com os *headers* CORS (Access-Control-Allow-Origin, Access-Control-Allow-Methods, etc.) para indicar se o domínio de origem é permitido. O API Gateway pode ser configurado para tratar a requisição OPTIONS.

---

 ## **11\. Tipos de API no API Gateway**

O API Gateway suporta três tipos principais de API, sendo as APIs REST as mais ricas em recursos e as HTTP as mais econômicas.

### **Comparativo de APIs (Alto Nível para o Exame)**

| Tipo de API | Características Principais | Uso Típico |
| :---- | :---- | :---- |
| **REST API** | **Rica em recursos.** Suporta Mapeamento de Dados (VTL), Plans de Uso, Chaves de API, Caching, WAF, Políticas de Recurso. | APIs complexas que exigem transformação de dados, segurança granular e monetização/limitação de acesso. |
| **HTTP API** | **Baixa latência e custo-efetiva.** Suporta apenas integrações **Proxy** (Lambda Proxy, HTTP Proxy). **Não** suporta Plans de Uso ou Chaves de API. | Alternativa mais simples e **muito mais barata** para APIs que exigem apenas roteamento direto e baixa latência. |
| **WebSocket API** | Comunicação **interativa bidirecional** (*Two-Way Communication*) e persistente entre cliente e servidor. | Aplicações em tempo real (chat, jogos multiplayer, plataformas de negociação financeira). |

---

## **12\. WebSocket APIs**

WebSockets permitem **comunicação bidirecional interativa** e **stateful** (com estado) ao estabelecer uma conexão persistente entre o cliente (navegador) e o servidor.

### **12.1 Fluxo da Conexão e Desconexão**

1. **Conexão Persistente:** O cliente se conecta ao URL do WebSocket (protocolo wss://).  
2. **$connect Route:** Ao estabelecer a conexão, uma rota especial ($connect) invoca uma **função Lambda** (onConnect). Essa Lambda é usada para **persistir o ID da Conexão** (geralmente no DynamoDB) e armazenar metadados do usuário. O ID da Conexão é persistente enquanto o cliente estiver conectado.  
3. **Transferência de Dados:** O cliente envia mensagens (**frames**) através dessa conexão persistente.  
4. **$disconnect Route:** Ao fechar a conexão, uma rota especial ($disconnect) invoca uma Lambda (onDisconnect) para limpar o ID da Conexão do banco de dados.

### **12.2 Comunicação Servidor $\\rightarrow$ Cliente (Two-Way)**

A comunicação do servidor (API Gateway) de **volta para o cliente** é feita através de uma **URL de Callback de Conexão** específica para aquele cliente.

* **Ação:** Uma função Lambda de *backend* deve fazer um **HTTP POST assinado (SigV4)** para a URL de *callback* (/connections/{connectionId}).  
* **Operações:** O *backend* pode usar essa URL de *callback* para:  
  * **POST:** Enviar uma mensagem para o cliente conectado.  
  * **GET:** Obter o *status* da conexão do cliente.  
  * **DELETE:** Desconectar o cliente.

### **12.3 Roteamento (Routing)**

* Como a conexão é persistente, o API Gateway usa **Routing** para determinar qual *backend* (ex: Lambda) deve ser invocado quando o cliente envia uma mensagem.  
* **Expressão de Seleção de Rota:** Você define uma expressão (ex: $request.body.action) para extrair um valor de um campo JSON da mensagem de entrada.  
* **Tabela de Rotas:** O valor extraído é comparado com as **Route Keys** definidas no API Gateway (ex: join, quit). As rotas mandatórias são connect, disconnect e default.  
* **$default Route:** Usada se nenhuma outra rota for correspondida.

---

## **13\. Arquitetura de Microservices com API Gateway**

O API Gateway atua como uma **interface unificada e centralizada** para múltiplas aplicações ou *backends* (Microservices).

### **13.1 Casos de Uso**

* **URL Única:** Fornece um único URL externo para clientes, **escondendo a complexidade** e a variedade de serviços internos (Load Balancers, EC2, S3, ECS, Lambda, etc.).  
* **Roteamento:** Diferentes rotas do API Gateway (ex: /service1, /docs) podem ser roteadas para diferentes *backends* (ex: ELB, S3, ASG).  
* **Domínios Customizados:** Permite registrar **domínios customizados** (*vanity URLs*) via Route 53 (ex: customer1.example.com), aplicando certificados SSL (via AWS Certificate Manager \- ACM) e roteando internamente para a API.  
* **Transformação Centralizada:** Permite aplicar regras de **encaminhamento e transformação** (usando VTL) antes de enviar a requisição para o *backend* real, garantindo que os *backends* recebam os dados no formato esperado.











