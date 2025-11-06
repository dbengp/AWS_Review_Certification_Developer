# **📑Soluções para Aplicações em Containers Docker na AWS com Amazon ECS, ECR e EKS**

## 1. Conceito e Objetivo Principal
* A solução conteinerizada da AWS (ECS, EKS e ECR) visa fornecer um ambiente escalável, seguro e totalmente gerenciado para empacotar, implantar e executar aplicações usando contêineres Docker. O Conceito central é abstrair a complexidade do gerenciamento de infraestrutura (servidores, sistemas operacionais, orquestração e patching), permitindo que os desenvolvedores foquem exclusivamente na lógica da aplicação.

### **2\. Tipos de Lançamento (Launch Types)**

| Característica | Fargate (Serverless) | EC2 (Gerenciado pelo Usuário) |
| :---- | :---- | :---- |
| **Gerenciamento de Infraestrutura** | AWS gerencia (Servidor *sempre* gerenciado). | Você provisiona, gerencia e faz *patch* nas instâncias EC2. |
| **Recurso de *Backend*** | Capacidade abstrata (pagamento por vCPU/RAM da Tarefa). | Instâncias EC2 provisionadas no seu ASG (Auto Scaling Group). |
| **Networking (Rede)** | **Obrigatório** o modo awsvpc. Cada **Tarefa** recebe seu próprio ENI e IP privado. | Permite mapeamento de portas dinâmico (hostPort: 0). |
| **Escalabilidade** | Aumente ou diminua o **número de Tarefas**. Simples, nativo. | Requer o escalonamento do **ASG** subjacente ou **Cluster Capacity Provider (Recomendado)**. |
| **Recomendação da AWS/Exame** | **Recomendado** para simplicidade e *serverless*. | Para requisitos específicos (GPUs, licenças, controle de SO/infra). |

🔑 **Nota de Exame:** A AWS e o exame *adoram* o **Fargate** por ser *serverless* e reduzir a sobrecarga operacional. Use Fargate a menos que o cenário exija controle sobre a VM.

### **3\. Funções do IAM no ECS (Segurança Crítica)**

O exame exige que você entenda a distinção entre as funções:

| Função | Aplicação | Uso Principal |
| :---- | :---- | :---- |
| **Task Role** | Aplicável a **Fargate e EC2**. Definida na **Definição de Tarefa**. | Permissões para o **código dentro do container** interagir com outros serviços AWS (ex: S3, DynamoDB, RDS). |
| **Task Execution Role** | Aplicável a **Fargate e EC2**. | Permissões para o **Agente ECS** (ou AWS Fargate) realizar ações, como: puxar a imagem do **ECR**, enviar logs para o **CloudWatch**, buscar segredos no **Secrets Manager/SSM Parameter Store**. |
| **EC2 Instance Profile** | **Apenas para EC2 Launch Type**. | Usado pelo **Agente ECS** em si para registrar a instância no Cluster e fazer chamadas de API do serviço ECS. |

### **4\. Definições de Tarefa (Task Definitions)**

* **Estrutura:** Um *blueprint* em JSON/YAML que define um ou mais contêineres (até 10).  
* **Campos Cruciais:** image (imagem do Docker), portMappings (mapeamento de portas), memory, cpu, environmentVariables, taskRoleArn, executionRoleArn.  
* **Mapeamento de Portas com ALB:**  
  * **EC2:** Use mapeamento de portas dinâmico ("hostPort": 0). O ALB, quando vinculado ao Serviço ECS, é inteligente o suficiente para descobrir a porta aleatória em que a Tarefa foi executada (via **Dynamic Host Port Mapping**).  
  * **Fargate:** Cada Tarefa tem seu próprio IP (awsvpc), então o ALB se conecta à porta do contêiner diretamente (ex: Porta 80).  
* **Variáveis de Ambiente:** Podem ser codificadas, ou carregadas dinamicamente via **SSM Parameter Store** ou **Secrets Manager** (para dados sensíveis), ou ainda via *bulk* (arquivo) do **Amazon S3**.

### **4.1\. Definições de Tarefa (JSON) para Fargate vs. EC2**

No Fargate, o foco é a definição da **capacidade** na própria tarefa, e o modo de rede é sempre **awsvpc**.
### **JSON da Definição de Tarefa (Fargate)**
```
{  
  "family": "web-app-fargate",  
  "requiresCompatibilities": [  
    "FARGATE"  
  ],  
  "networkMode": "awsvpc",  
  "cpu": "256",  
  "memory": "512",  
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",  
  "taskRoleArn": "arn:aws:iam::123456789012:role/webAppTaskRole",  
  "containerDefinitions": [  
    {  
      "name": "web-app-container",  
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/web-app:latest",  
      "portMappings": [  
        {  
          "containerPort": 80,  
          "protocol": "tcp"  
        }  
      ],  
      "logConfiguration": {  
        "logDriver": "awslogs",  
        "options": {  
          "awslogs-group": "/ecs/web-app-fargate",  
          "awslogs-region": "us-east-1",  
          "awslogs-stream-prefix": "ecs"  
        }  
      }  
    }  
  ]  
}
```
### **Pontos Chave para o Exame (Fargate):**

1. **requiresCompatibilities: \["FARGATE"\]**  
   * **Indica o tipo de lançamento.**  
2. **networkMode: awsvpc**  
   * **Obrigatório para Fargate.** Cada Tarefa recebe seu próprio ENI (Elastic Network Interface) e IP privado.  
3. **cpu e memory (Nível da Tarefa)**  
   * **Obrigatório para Fargate.** Você deve alocar os recursos diretamente na Definição de Tarefa. A AWS provisiona a infraestrutura sob medida com base nesses valores.  
4. **portMappings (Apenas containerPort)**  
   * Não há necessidade de hostPort (porta do host) porque a tarefa tem seu próprio IP.

No EC2, você tem mais flexibilidade de rede, e a Tarefa é executada em uma instância que você gerencia. O foco está no mapeamento dinâmico de portas.

### **JSON da Definição de Tarefa (EC2)**
```
{  
  "family": "web-app-ec2",  
  "requiresCompatibilities": [  
    "EC2"  
  ],  
  "networkMode": "bridge",  
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",  
  "taskRoleArn": "arn:aws:iam::123456789012:role/webAppTaskRole",  
  "containerDefinitions": [  
    {  
      "name": "web-app-container",  
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/web-app:latest",  
      "cpu": 0,  
      "memoryReservation": 128,  
      "portMappings": [  
        {  
          "containerPort": 80,  
          "hostPort": 0,  
          "protocol": "tcp"  
        }  
      ],  
      "logConfiguration": {  
        "logDriver": "awslogs",  
        "options": {  
          "awslogs-group": "/ecs/web-app-ec2",  
          "awslogs-region": "us-east-1",  
          "awslogs-stream-prefix": "ecs"  
        }  
      }  
    }  
  ]  
}
```
### **Pontos Chave para o Exame (EC2):**

1. **requiresCompatibilities: \["EC2"\]**  
   * **Indica o tipo de lançamento.**  
2. **networkMode: bridge**  
   * Um modo comum para o EC2, onde o Agente ECS gerencia o mapeamento de portas na instância (o modo **host** também é possível).  
3. **cpu e memory (Nível do Contêiner)**  
   * Os recursos são alocados no nível do contêiner (containerDefinitions) e retirados do total disponível da instância EC2.  
   * **memoryReservation**: Memória garantida para o contêiner.  
4. **portMappings: containerPort e hostPort**  
   * **"hostPort": 0**: Este é o padrão para o **Mapeamento Dinâmico de Portas**. Isso diz ao Agente ECS para selecionar uma porta efêmera aleatória na instância EC2 disponível.  
   * **Integração com ALB:** O Application Load Balancer é o único que sabe consultar o ECS para descobrir qual porta aleatória foi atribuída a uma Tarefa específica, o que permite rodar múltiplas cópias da mesma Tarefa (todas expondo a porta 80 do contêiner) na mesma instância EC2.

---

### **Resumo das Diferenças Chave:**

| Configuração no JSON | Fargate | EC2 |
| :---- | :---- | :---- |
| **Tipo de Lançamento** | "requiresCompatibilities": \["FARGATE"\] | "requiresCompatibilities": \["EC2"\] |
| **Alocação de Recurso** | cpu e memory no nível da **Tarefa** (obrigatório). | cpu e memoryReservation no nível do **Contêiner**. |
| **Modo de Rede** | "networkMode": "awsvpc" (obrigatório). | "networkMode": "bridge" ou "host" (comum). |
| **Mapeamento de Portas** | Apenas "containerPort". | "containerPort" e "hostPort": 0 (dinâmico) para uso com ALB. |

Entender o **hostPort: 0** e o **networkMode: awsvpc** é a chave para distinguir os dois tipos de lançamento nas questões da prova.

### **5\. Armazenamento e Persistência de Dados**

* **Compartilhamento de Dados na Mesma Tarefa (Sidecar):** Utilize **Bind Mounts** (Volumes de Dados efêmeros) dentro da mesma Definição de Tarefa. Permite que contêineres como o *sidecar* de logs e o contêiner de aplicação compartilhem um sistema de arquivos local (efêmero: vinculado ao ciclo de vida da Tarefa/Instância).  
* **Armazenamento Persistente Multi-AZ (Definitivo):** Use o **Amazon EFS (Elastic File System)**.  
  * **Vantagem:** É um sistema de arquivos de rede **serverless** (pago conforme o uso) que é compatível com os tipos de lançamento **EC2 e Fargate**.  
  * **Caso de Uso:** Permite o **armazenamento compartilhado persistente multi-AZ** entre várias Tarefas ECS.

### **6\. Auto Scaling (Dimensionamento Automático)**

| Componente a ser Escalado | Serviço Utilizado | Tipo de Lançamento | Métricas Chave |
| :---- | :---- | :---- | :---- |
| **Tarefas do Serviço ECS** | **Application Auto Scaling** | Fargate e EC2 | Utilização de CPU/Memória do **Serviço** ou Contagem de Requisições do **ALB** por destino. |
| **Instâncias EC2 do Cluster** | **Cluster Capacity Provider (Recomendado)** | Apenas EC2 | Mais inteligente: Escala o ASG automaticamente quando o ECS precisa de mais recursos para iniciar Tarefas. |
| **Instâncias EC2 do Cluster** | **Auto Scaling Group Scaling (Tradicional)** | Apenas EC2 | Escala o ASG com base em métricas como Utilização de CPU da **Instância**. |

🔑 **Nota de Exame:** Ao escalar o serviço (número de Tarefas), você usa o Application Auto Scaling. Ao escalar a infraestrutura subjacente (Instâncias EC2), use o **Cluster Capacity Provider**.

### **7\. Estratégias de Colocação de Tarefas (Apenas para ECS no EC2)**

Quando o ECS precisa decidir onde colocar (ou remover) uma Tarefa no EC2, ele usa Estratégias e Restrições:

| Estratégia | Objetivo | JSON Key | Descrição |
| :---- | :---- | :---- | :---- |
| **Binpack** | **Economia de Custos**. Empacota o máximo de Tarefas em uma Instância antes de ir para a próxima. | binpack:memory ou binpack:cpu | Minimiza o número de instâncias EC2 em uso. |
| **Spread (Distribuição)** | **Alta Disponibilidade**. Distribui uniformemente as Tarefas entre valores especificados (ex: AZ ou ID da Instância). | spread:availabilityZone | Maximiza a distribuição e a resiliência. |
| **Random** | Coloca Tarefas aleatoriamente. (Não é ideal). | N/A | Simplesmente aleatório, sem lógica. |

| Restrição | Objetivo | JSON Key | Exemplo de Uso |
| :---- | :---- | :---- | :---- |
| **DistinctInstance** | Garante que cada Tarefa seja colocada em uma Instância de Contêiner diferente. | distinctInstance | Nunca ter duas Tarefas do mesmo serviço na mesma VM. |
| **MemberOf** | Coloca a Tarefa em instâncias que satisfaçam uma expressão específica. | memberOf:instanceType \== t2.micro | Forçar a colocação em instâncias de um determinado tipo. |

### **8\. Estratégias de Implantação (Rolling Updates)**

Ao atualizar um Serviço ECS (v1 para v2), você controla a transição via **Rolling Update** usando dois parâmetros:

* **minimumHealthyPercent:** O percentual **mínimo** de Tarefas em execução (v1 ou v2) que deve estar em **funcionamento** durante o processo de implantação.  
  * *Exemplo:* 50% permite que o serviço encerre tarefas antigas antes de iniciar as novas, o que pode resultar em capacidade temporária reduzida.  
* **maximumPercent:** O percentual **máximo** de Tarefas em execução (v1 e v2 combinadas) permitido.  
  * *Exemplo:* 200% significa que o serviço pode ter o dobro do número desejado de tarefas em execução durante a implantação (mantendo a capacidade total e iniciando as novas antes de encerrar as antigas).

### **9\. Arquiteturas de Solução (Integração)**

O ECS se integra a outros serviços AWS para fluxos de trabalho *serverless*:

* **Event-Driven (Orientado a Eventos):**  
  * **Amazon S3 & EventBridge:** O EventBridge pode ser acionado por eventos do S3 (upload de objeto) e, em seguida, **Executar uma Tarefa ECS (Fargate)** para processar o objeto, usando o Task Role para acessar o S3 e o DynamoDB.  
* **Job Agendado (Cron):**  
  * **EventBridge Schedule:** Agenda uma regra para **Executar uma Tarefa ECS (Fargate)** a cada intervalo (ex: a cada 1 hora) para processamento em lote.  
* **Processamento de Fila (SQS):**  
  * **SQS & ECS Auto Scaling:** O Serviço ECS consome mensagens do SQS. O **Service Auto Scaling** pode ser configurado para aumentar ou diminuir o número de Tarefas com base no **número de mensagens visíveis no SQS** (métricas do CloudWatch), gerenciando a *backlog* da fila.  
* **Monitoramento de Cluster:**  
  * **EventBridge:** O EventBridge pode **interceptar eventos do ciclo de vida do ECS** (ex: mudança de estado da Tarefa para "STOPPED") e acionar ações, como enviar alertas para o **SNS**.


### **10\. Amazon Elastic Container Registry (ECR)**

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
