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
O **Amazon ECR (Elastic Container Registry)** é um **registro de imagens Docker totalmente gerenciado e seguro** que facilita o armazenamento, o gerenciamento e a implantação de imagens de contêineres. É o "Docker Hub" privado (ou público) da AWS.

| Conceito | Descrição | Foco na Prova |
| :---- | :---- | :---- |
| **Integração NATIVA** | O ECR se integra perfeitamente ao **Amazon ECS** e ao **EKS**, facilitando o *pull* de imagens. | Imagens armazenadas no ECR são acessadas por **Task Execution Role** (Fargate/EC2) ou **EC2 Instance Profile** (EC2). |
| **Segurança** | Todo o acesso é controlado pelo **IAM**. As imagens são criptografadas em repouso. | A autenticação é feita via token de autorização. O controle de acesso é via políticas IAM. |
| **Armazenamento** | As imagens são armazenadas internamente no **Amazon S3**, o que garante alta durabilidade e disponibilidade. | Não é necessário gerenciar o armazenamento subjacente (gerenciado pela AWS). |
| **Tipos de Repositório** | **Private Registry** (padrão, acesso restrito à sua conta/região) e **Public Registry** (para publicar e compartilhar imagens publicamente). | O foco do desenvolvedor é o **Private Registry**. |
| **Recursos Avançados** | Suporte a **Scanning de Vulnerabilidades**, **Gerenciamento do Ciclo de Vida da Imagem** (para limpar imagens antigas/não *tagged*) e **Tagging de Imagem** (controle de versão). | *Scanning* de vulnerabilidade ajuda a identificar problemas de segurança *antes* da implantação. |

### **10.1\. O Fluxo de Trabalho (Developer Lifecycle) e Comandos CLI**

O exame de Desenvolvedor exige que você entenda a sequência de comandos para levar uma imagem do seu ambiente local para o ECR e, por fim, para o ECS.

#### **Etapa 1: Autenticar e Obter Acesso ao ECR**

Antes de tudo, o cliente Docker local precisa de um token de autenticação temporário para se comunicar com o ECR.

| Ação | Comando AWS CLI |
| :---- | :---- |
| **Obter o Token de Autorização** | \`aws ecr get-login-password \--region |
| **Exemplo** | \`aws ecr get-login-password \--region us-east-1 |
| **Importância:** | Este comando gera um token de curto prazo e o envia diretamente para o *login* do Docker. **Se for um pipeline CI/CD ou um script, este é o comando fundamental.** |

#### **Etapa 2: Criar e "Taggear" (Rotular) a Imagem Docker**

Você precisa construir sua imagem e, em seguida, rotulá-la com o URI (Uniform Resource Identifier) completo do ECR.

| Ação | Comando Docker CLI |
| :---- | :---- |
| **Construir a Imagem** | docker build \-t \<nome-local\> . |
| **Taggear a Imagem (ECR)** | docker tag \<nome-local\>:latest \<URI-do-ECR\>/\<repositorio\>:\<tag\> |
| **Exemplo** | docker tag web-app:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/minha-app:v1.0.0 |
| **Importância:** | O **Tagging** correto garante que o Docker saiba para onde enviar a imagem no próximo passo. O URI é crucial: Conta.dkr.ecr.Regiao.amazonaws.com/Repositório:Tag. |

#### **Etapa 3: Enviar (Push) a Imagem para o ECR**

O *push* envia a imagem para o seu repositório ECR privado.

| Ação | Comando Docker CLI |
| :---- | :---- |
| **Fazer o Push da Imagem** | docker push \<URI-do-ECR\>/\<repositorio\>:\<tag\> |
| **Exemplo** | docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/minha-app:v1.0.0 |
| **Importância:** | A imagem está agora armazenada de forma segura e criptografada (pelo KMS ou S3 SSEC) no ECR, pronta para ser consumida pelo ECS/EKS. |

#### **Etapa 4: Configurar o ECS para Puxar (Pull) a Imagem**

Este passo não usa comandos CLI de Docker, mas é crucial para o exame, pois envolve **IAM**.

| Configuração | Onde se Define | Requisito de Permissão (IAM) |
| :---- | :---- | :---- |
| **URI da Imagem** | Na **Definição de Tarefa** do ECS (campo image). | N/A (É apenas uma string) |
| **Permissão para Pull** | Na **Task Execution Role** (para Fargate e EC2) ou **EC2 Instance Profile** (para EC2). | Políticas ecr:GetAuthorizationToken, ecr:BatchCheckLayerAvailability, ecr:GetDownloadUrlForLayer e ecr:BatchGetImage. |
| **Importância:** | O ECS **NÃO** usa sua chave de usuário (aws configure). Ele usa a **Task Execution Role** para se autenticar automaticamente e buscar o token, garantindo que a Tarefa (ou o Agente ECS) tenha permissão para fazer o *pull*. |  |

### **10.2\. Gerenciamento e Segurança com o ECR**

* **Verificação de Vulnerabilidade (Image Scanning):** O ECR pode escanear as imagens (usando a base de dados do Common Vulnerabilities and Exposures \- CVE) para identificar vulnerabilidades. O exame pode perguntar sobre a importância de executar esse scan **antes** de implantar a imagem em produção.  
* **Políticas de Ciclo de Vida (Lifecycle Policies):** Você pode definir regras para expirar ou limpar automaticamente imagens antigas.  
  * *Exemplo:* Manter apenas as últimas 10 imagens ou excluir imagens com mais de 90 dias que não tenham uma *tag* específica. **Isso é crucial para otimizar custos e manter o repositório limpo.**  
* **Permissões de Repositório:** Embora o IAM controle o acesso do usuário, você pode aplicar uma **Policy de Repositório** diretamente ao ECR para permitir o acesso *cross-account* (de outra conta AWS) ou restringir ações específicas a um *role* ou usuário.

### **11\. Amazon Elastic Kubernetes Service (EKS)**
O **Amazon EKS** é um serviço gerenciado que permite executar clusters **Kubernetes de código aberto** na AWS. Ele é uma alternativa ao ECS, oferecendo a portabilidade e a padronização da API do Kubernetes.
| Componente Kubernetes | Analogia ECS (para comparação) | Descrição |
| :---- | :---- | :---- |
| **Control Plane (Plano de Controle)** | *Gerenciado pela AWS* | O "cérebro" do cluster (API Server, etcd, Scheduler, Controller Manager). **A AWS gerencia, dimensiona, faz *patch* e garante a Alta Disponibilidade (HA)**. |
| **Worker Nodes (Nós de Trabalho)** | Instâncias EC2 (ou Fargate) | A infraestrutura onde os Pods são executados. |
| **Pod** | Tarefa (Task) | A menor unidade implantável. Um Pod é uma abstração que encapsula um ou mais containers (Docker). |
| **Service (Serviço)** | Serviço ECS | Define uma política lógica para acessar um grupo de Pods. Geralmente integrado a um ALB ou NLB. |

### **11.1\. Modos de Lançamento (Compute Options)**

O EKS oferece a mesma flexibilidade de infraestrutura que o ECS, mas com a API do Kubernetes:

| Modo de Lançamento | Gerenciamento do Nó | Caso de Uso |
| :---- | :---- | :---- |
| **EKS Fargate** | **Serverless (Sem Nós)**. AWS gerencia totalmente a infraestrutura. | Simplicidade, cargas de trabalho que não precisam de controle fino sobre o nó, ambientes *burst*. |
| **Managed Node Groups (Recomendado)** | AWS provisiona e gerencia o **Auto Scaling Group (ASG)** e o ciclo de vida dos Nós EC2. | Equilíbrio entre controle e gerenciamento. Suporta instâncias On-Demand e Spot. |
| **Self-Managed Nodes** | Você provisiona e gerencia o ASG e o ciclo de vida dos Nós EC2 (maior controle, maior esforço). | Requisitos de customização do SO, *host* dedicado ou configurações de rede muito específicas. |

### **11.2\. Networking e Segurança (Integração AWS)**

* **VPC e CNI (Container Network Interface):** O EKS usa o **AWS VPC CNI** para atribuir um IP privado da sua VPC diretamente a cada **Pod**. Isso permite que os Pods interajam com outros recursos AWS (RDS, S3) sem *NAT*.  
* **Load Balancing:**  
  * **Kubernetes Service (Type LoadBalancer):** Pode provisionar um **NLB** (Network Load Balancer) ou **ALB** (Application Load Balancer) para expor os Pods.  
  * **AWS Load Balancer Controller (Adicionalmente):** Permite usar um Ingress do Kubernetes para provisionar um ALB, oferecendo recursos L7 (HTTPS, roteamento baseado em *path*).  
* **IAM para Pods (IAM Roles for Service Accounts \- IRSA):**  
  * **Crucial para o Exame\!** No EKS, a maneira segura e recomendada de dar permissão AWS a um **Pod** (e não ao Nó inteiro) é através do **IRSA**.  
  * O IRSA permite que as Service Accounts do Kubernetes assumam um **IAM Role**, garantindo que apenas os Pods associados àquela Service Account tenham permissão para interagir com os serviços AWS (ex: S3, DynamoDB).  
  * *Lembrete:* No ECS, isso é feito pela **Task Role**.

### **11.3\. Armazenamento Persistente (CSI)**

* **CSI (Container Storage Interface):** É a interface padronizada que permite que o Kubernetes se integre a diferentes tecnologias de armazenamento. O EKS exige o uso de **CSI Drivers** compatíveis.  
* **Opções de Armazenamento:**  
  * **Amazon EBS:** Usado para armazenamento em bloco persistente para Pods (geralmente acesso de um único Pod).  
  * **Amazon EFS:** Única opção de armazenamento persistente multi-AZ que funciona com o **EKS Fargate**. Ideal para volumes que precisam ser compartilhados entre múltiplos Pods.

### **11.4\. Comandos e Fluxo de Trabalho CLI**

O fluxo de trabalho do desenvolvedor no EKS é centrado em duas ferramentas CLI: **aws** e **kubectl**.

#### **🔑 Fluxo Essencial: Conectar e Implantar**

| Objetivo | Comando CLI | Foco do Exame |
| :---- | :---- | :---- |
| **Conectar o kubectl ao EKS** | aws eks update-kubeconfig \--name \<cluster-name\> \--region \<regiao\> | **Crucial:** Este comando obtém credenciais temporárias do IAM e atualiza o arquivo \~/.kube/config, permitindo que você use o kubectl. |
| **Verificar o Status do Cluster** | kubectl get svc, kubectl get nodes, kubectl get pods \-n kube-system | Usado para diagnóstico e verificação de *deployments*. |
| **Implantar um Recurso (Deployment, Service)** | kubectl apply \-f \<manifesto.yaml\> | O comando fundamental para aplicar a configuração de Kubernetes. |
| **Ver logs de um Pod** | kubectl logs \<pod-name\> | Essencial para *troubleshooting* de aplicações. |

#### **🛠️ Exemplo Prático de Deploy**

1. **Crie o arquivo de manifesto** (deployment.yaml) que define o Pod (referenciando uma imagem do ECR) e o *deployment*.  
2. **Use o AWS CLI** para garantir que o **IAM** e a configuração estejam corretos (update-kubeconfig).  
3. **Use o Kubectl** para implantar (kubectl apply \-f deployment.yaml).

💡 **Nota de Prova:** O Kubernetes é **agnóstico de nuvem**, o que significa que sua API é a mesma em qualquer provedor. A vantagem do EKS é a gerência do Control Plane e a integração nativa com o **IAM (IRSA)**, **VPC CNI** e serviços de armazenamento **(EFS/EBS)**.

### **11.5\. Segurança e Exposição de Serviço**

#### **1\. IAM Roles for Service Accounts (IRSA) \- O Ponto Chave da Segurança EKS**

O **IRSA** é a maneira recomendada e mais segura de conceder permissões da AWS a workloads (Pods) executados no EKS.

#### **Problema que o IRSA Resolve:**

Tradicionalmente, para dar acesso a um serviço AWS (ex: S3) a um Pod, você teria que anexar a *IAM Role* de permissão à **instância EC2 (Worker Node)**. Isso significa que **todos os Pods** que rodam naquela instância herdariam a mesma permissão, o que viola o princípio do **menor privilégio**.

#### **Como o IRSA Funciona:**

1. **Criação de um OIDC Provider:** O cluster EKS é configurado com um OpenID Connect (OIDC) Provider, permitindo que o IAM confie nas *Service Accounts* (SAs) do Kubernetes.  
2. **Criação da Service Account (SA):** No Kubernetes, você define uma SA, que será a identidade do seu Pod.  
3. **Criação da IAM Role:** No IAM, você cria uma *Role* com a política de permissão necessária (ex: s3:GetObject).  
4. **Associação:** Você configura uma "trust relationship" na IAM Role, permitindo que a SA específica do Kubernetes (a partir do OIDC Provider do cluster EKS) assuma essa *Role*.  
5. **Anotação no Pod:** O *Deployment* ou *Pod* é anotado para usar a SA, vinculando o Pod à IAM Role.

**Resultado:** Apenas os Pods que usam aquela *Service Account* específica recebem as credenciais temporárias do AWS para acessar os serviços, garantindo o menor privilégio.

#### **📝 Exemplo (Abstraído)**

No manifesto YAML do Kubernetes (na seção spec do Pod):
```
apiVersion: v1  
kind: ServiceAccount  
metadata:  
  name: s3-reader-sa  
  annotations:  
    \# A anotação mais importante que vincula a SA ao IAM Role  
    "eks.amazonaws.com/role-arn": "arn:aws:iam::123456789012:role/S3ReaderRole" 

\# ... (Seu Deployment/Pod usando esta ServiceAccount)
```

### **2\. Exposição de Serviço (Service Type LoadBalancer)**

Para expor uma aplicação em um Pod do EKS à Internet ou à sua VPC, você usa o recurso **Service** do Kubernetes. Ao usar o tipo LoadBalancer, o AWS **provisiona automaticamente** um Application Load Balancer (ALB) ou Network Load Balancer (NLB).

#### **📝 Manifesto YAML para Expor com um NLB**

Se o **AWS Load Balancer Controller** (que provisiona ALBs via Ingress) não estiver instalado, um Service de tipo LoadBalancer geralmente provisiona um **NLB** (Network Load Balancer) por padrão (camada 4).
```
apiVersion: v1  
kind: Service  
metadata:  
  name: minha-web-service  
  labels:  
    app: minha-web-app  
  annotations:  
    \# Opcional, mas útil: Força a criação de um NLB (Layer 4\) interno (sem acesso público)  
    \# "service.beta.kubernetes.io/aws-load-balancer-internal": "true"   
    \# Opcional, mas útil: Define que o provisionamento deve ser um NLB (Network Load Balancer)  
    \# "service.beta.kubernetes.io/aws-load-balancer-type": "nlb"   
spec:  
  selector:  
    app: minha-web-app \# Deve corresponder aos labels dos seus Pods/Deployment  
  type: LoadBalancer    \# \<-- Tipo que diz ao EKS/AWS para criar um ELB/NLB  
  ports:  
    \- protocol: TCP  
      port: 80         \# Porta que o Load Balancer ouve  
      targetPort: 8080 \# Porta que o Pod/Container está exposto
```
#### **📝 Manifesto YAML para Expor com um ALB (via Ingress)**

O uso de um **Application Load Balancer (ALB)**, com recursos L7 (roteamento baseado em *path*, regras de host), é feito através de um recurso **Ingress**. Isso requer que o **AWS Load Balancer Controller** esteja instalado e rodando no seu cluster.

```
apiVersion: networking.k8s.io/v1  
kind: Ingress  
metadata:  
  name: meu-alb-ingress  
  annotations:  
    \# Anotações importantes que instruem o controlador a criar um ALB  
    "kubernetes.io/ingress.class": "alb"  
    "alb.ingress.kubernetes.io/scheme": "internet-facing" \# Público ou interno  
    "alb.ingress.kubernetes.io/target-type": "ip"        \# O Pod é o destino (Fargate usa 'ip')  
    "alb.ingress.kubernetes.io/listen-ports": '\[{"HTTP": 80}\]'  
spec:  
  rules:  
  \- http:  
      paths:  
      \- path: /  
        pathType: Prefix  
        backend:  
          service:  
            name: minha-web-service \# Nome do Service k8s (Type: NodePort ou ClusterIP)  
            port:  
              number: 80
```
### **🎯 Ponto-chave para a Prova:**

* **Segurança no Pod:** Use **IRSA**.  
* **Exposição Simples (L4):** Use Service de type: LoadBalancer.  
* **Exposição Avançada (L7):** Use **Ingress** (e verifique as anotações do AWS ALB Controller).

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
