## **📑Soluções para Aplicações em Containers Docker na AWS com Amazon ECS, ECR e EKS**

## 1. Conceito e Objetivo Principal
* A solução conteinerizada da AWS (ECS, EKS e ECR) visa fornecer um ambiente escalável, seguro e totalmente gerenciado para empacotar, implantar e executar aplicações usando contêineres Docker. O Conceito central é abstrair a complexidade do gerenciamento de infraestrutura (servidores, sistemas operacionais, orquestração e patching), permitindo que os desenvolvedores foquem exclusivamente na lógica da aplicação.

### **2\. Tipos de Lançamento do ECS**

O Amazon Elastic Container Service (ECS) oferece duas abordagens para gerenciar a infraestrutura subjacente:

| Tipo de Lançamento | Foco em Certificação |
| :---- | :---- |
| **EC2 Launch Type** | O usuário gerencia instâncias EC2 e o Agente ECS. Oferece controle total sobre o SO e recursos. |
| **Fargate Launch Type** | **Serverless (Sem Servidor).** O AWS gerencia o backend. Foco em **Custo** (paga-se por recursos alocados) e **Segurança** (isolamento de host superior). |

#### **O Agente ECS e Suas Configurações (EC2 Launch Type)**

O **Agente ECS** é o software obrigatório executado nas instâncias EC2, configurado principalmente pela variável **ECS\_CLUSTER**.

#### **Comandos Essenciais AWS CLI para ECS**

| Comando | Função | Exemplo |
| :---- | :---- | :---- |
| **aws ecs run-task** | Inicia uma Tarefa ECS **avulsa** (não gerenciada por um Serviço). Ideal para tarefas pontuais ou *batch*. | aws ecs run-task \--cluster \<nome\> \--task-definition \<def\> |
| **aws ecs update-service** | Atualiza um Serviço ECS existente (ex: mudar a contagem desejada de Tarefas, mudar a Definição de Tarefa, ou forçar uma nova implantação). | aws ecs update-service \--cluster \<nome\> \--service \<serviço\> \--desired-count 5 |
| **aws ecs describe-services** | Obtém informações detalhadas sobre o estado atual de um ou mais serviços ECS, incluindo métricas de saúde e eventos. | aws ecs describe-services \--cluster \<nome\> \--services \<serviço\> |

### **3\. Gerenciamento de Identidade e Acesso (IAM)**

* **Perfil de Instância EC2:** Usado pelo **Agente ECS**.  
* **Função de Tarefa ECS:** Concede permissões diretamente ao **código da aplicação**.

### **4\. Integração com Balanceadores de Carga**

* **ALB:** **Mais suportado e recomendado** (Funciona com Fargate).  
* **NLB:** Para alta taxa de transferência.

### **5\. Armazenamento Persistente**

O **Amazon EFS** (Elastic File System) é a solução ideal, sendo Multi-AZ e Serverless, e permitindo montagem direta nas Tarefas.

### **6\. Escalabilidade e Application Auto Scaling**

* **Nível de Serviço (Tarefas):** Gerenciado pelo **AWS Application Auto Scaling** com métricas de CPU, Memória ou Requisições do ALB.  
* **Nível de Cluster (Infraestrutura EC2):** **Provedores de Capacidade (Capacity Providers)** é o método preferencial para escalar instâncias EC2 sob demanda.

### **7\. Estratégia de Atualização de Serviço (Rolling Update)**

Controlada por **Minimum Healthy Percent** e **Maximum Percent**. A configuração de Maximum Percent \> 100% garante **Zero Downtime**.

### **8\. Padrões Arquiteturais e Integrações do ECS**

* **EventBridge:** Acionamento de Tarefas ECS em resposta a eventos (ex: S3) ou para **Tarefas Agendadas**.  
* **SQS:** Permite que o **ECS Service Auto Scaling** dimensione o número de Tarefas com base na profundidade da fila.

### **9\. Amazon ECS Task Definitions (Em Profundidade)**

* **Mapeamento de Portas:** **Fargate** usa IP/ENI por Tarefa. **EC2** usa **Mapeamento Dinâmico** (hostPort: 0\) com ALB.  
* **Variáveis Confidenciais:** Devem ser referenciadas no **SSM Parameter Store** ou **Secrets Manager**.

### **10\. Estratégias e Restrições de Colocação de Tarefas**

Aplica-se **apenas ao EC2 Launch Type**: **Binpack** (Custo), **Spread** (Alta Disponibilidade), **distinctInstance** ou **memberOf**.

### **11\. Amazon Elastic Container Registry (ECR)**

O ECR armazena e gerencia imagens Docker, com **Verificação de Vulnerabilidade** e **Políticas de Ciclo de Vida**.

* **Autenticação CLI (Recomendada):** Token temporário via aws ecr get-login-password.

### **12\. Amazon Elastic Kubernetes Service (EKS)**

O EKS é o serviço gerenciado do **Kubernetes** na AWS.

* **Tipos de Nó:** Gerenciados, Autogerenciados ou **Fargate** (Serverless).  
* **Armazenamento:** **EFS** é a única classe de armazenamento que funciona com o Fargate no EKS.  
* **Comandos Essenciais CLI:**  
  * **aws eks update-kubeconfig**: Configura o acesso ao cluster.  
  * **kubectl get nodes** / **kubectl get pods** / **kubectl apply**: Comandos padrão do Kubernetes.

---

## **📘 Apêndice: Referência Rápida de Docker e Configurações Essenciais**

Este apêndice oferece uma visão geral das configurações de contêineres e comandos Docker relevantes no contexto AWS.

### **I. Conceitos Essenciais do Docker**

* **Contêiner:** Unidade de software portátil e executável.  
* **Imagem:** Template somente leitura para criar um contêiner (construído via **Dockerfile**).

### **II. Configurações Importantes para Consistência**

| Configuração | Função para Serviços na AWS |
| :---- | :---- |
| **HEALTHCHECK** | Comando no Dockerfile para verificar o estado interno e garantir que apenas contêineres "prontos" recebam tráfego (saúde do ALB/ECS). |
| **STOPSIGNAL** | Define o sinal de encerramento (SIGTERM padrão) para permitir o *graceful shutdown* da aplicação. |
| **LOG\_DRIVER** | O awslogs é o *driver* padrão para envio de logs ao **Amazon CloudWatch Logs**. |

### **III. Principais Comandos Docker e AWS CLI**

Comandos essenciais para desenvolvimento local e CI/CD:

* docker build / docker run / docker push  
* aws configure  
* aws ecr create-repository
