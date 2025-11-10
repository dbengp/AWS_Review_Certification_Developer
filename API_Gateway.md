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
