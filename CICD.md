# **🚀 CI/CD \- AWS**

## **1. Conceito e Objetivo Principal**

### **Integração Contínua (CI)**

É a prática onde os desenvolvedores enviam (*push*) código frequentemente para um repositório central. Um servidor de *build* (como o **CodeBuild** ou Jenkins) verifica, testa e integra o código assim que ele é enviado, garantindo que os *bugs* sejam encontrados e corrigidos rapidamente.

### **Entrega Contínua (CD)**

Após o sucesso do CI, o CD garante que o código testado seja implantado automaticamente em ambientes (como desenvolvimento, teste ou *staging*). Isso permite lançamentos mais rápidos e frequentes (ex: várias vezes ao dia).

### **Visão Geral dos Serviços AWS (Code Services)**

| Serviço | Função Principal | Status Atual |
| :---- | :---- | :---- |
| **CodeCommit** | Armazenamento de código-fonte Git privado e totalmente gerenciado. | **Descontinuado para novos clientes** (desde Jul/2024). A AWS recomenda migrar para soluções externas como GitHub ou GitLab. |
| **CodePipeline** | Orquestração visual e automação de todo o fluxo de CI/CD. | Essencial para orquestrar as fases. |
| **CodeBuild** | Compilação e teste do código em ambientes de contêiner. | Executa as instruções do *build* e testes. |
| **CodeDeploy** | Serviço de implantação que automatiza a atualização de aplicações em vários tipos de *targets*. | Faz o *deployment* final em EC2, Lambda ou ECS. |
| **CodeStar** | Ferramenta que unifica os serviços Code (CodeCommit, CodePipeline, etc.) para gerenciar projetos de desenvolvimento em um só lugar. | Simplifica a configuração inicial. |
| **CodeGuru** | Análise automatizada de código (revisão de código) e perfis de desempenho usando Machine Learning. | Adiciona segurança e otimização ao ciclo. |

---

## **2\. AWS CodeCommit (Controle de Versão)**

O CodeCommit fornecia um **repositório Git privado** e totalmente gerenciado na AWS.

* **Benefícios:** Sem limite de tamanho, altamente disponível, e código criptografado (*in-transit* e *at-rest* usando KMS).  
* **Segurança:** Utiliza comandos Git padrão, mas a autenticação é gerenciada pelo **IAM** (usuários e políticas para controle de acesso).  
* **Observação Crítica:** Devido à descontinuação para novos clientes (a partir de 25 de julho de 2024), a documentação da AWS agora recomenda o uso de soluções Git externas.

---

## **3\. AWS CodePipeline (Orquestração de CI/CD)**

O CodePipeline é o *workflow* visual que orquestra as diferentes fases do CI/CD, ligando serviços como CodeCommit, CodeBuild e CodeDeploy.

### **Estrutura do Pipeline**

1. **Source:** Onde o código-fonte está (ex: CodeCommit, GitHub, Bitbucket, ECR, S3).  
2. **Build:** Onde o código é compilado e testado (ex: CodeBuild, Jenkins).  
3. **Deploy:** Onde a aplicação é implantada (ex: CodeDeploy, Elastic Beanstalk, CloudFormation, ECS, S3).  
4. **Ações:** Cada **Stage** (fase) pode conter ações **sequenciais** e/ou **paralelas**.  
5. **Aprovação Manual:** Pode-se definir um passo de **aprovação manual** em qualquer *stage* (comum antes do *deployment* em produção).

### **Funcionamento Interno (Artifacts)**

* O CodePipeline usa um **bucket S3** interno para armazenar **Artifacts** (artefatos) – que são os arquivos ou a saída de uma fase.  
* O CodePipeline extrai o código-fonte, armazena-o como um *artifact* no S3, e depois **transfere** esse *artifact* para a próxima fase (ex: CodeBuild). O CodeBuild não precisa acessar o repositório diretamente, ele recebe os artefatos do CodePipeline via S3.

### **Monitoramento e *Troubleshooting***

* O **IAM Service Role** do CodePipeline deve ter permissões suficientes para interagir com todos os serviços envolvidos (CodeCommit, CodeBuild, S3, etc.).  
* Eventos de mudança de estado do *pipeline* (ex: falha) podem ser monitorados e notificados usando **EventBridge (CloudWatch Events)**.

---

## **4\. AWS CodeBuild (Compilação e Teste)**

O CodeBuild é um serviço totalmente gerenciado que executa a compilação, o empacotamento e os testes do código em ambientes de contêiner efêmeros.

### **buildspec.yml**

* As **instruções de *build*** devem ser definidas no arquivo **buildspec.yml**, que é a melhor prática e o foco do exame.  
* O arquivo buildspec.yml deve estar na **raiz do seu código**.

### **Componentes Chave do buildspec.yml**

* **environment:** Define variáveis de ambiente, que podem puxar valores de texto simples ou de serviços de segredos (como **SSM Parameter Store** ou **Secrets Manager**) para evitar armazenar senhas no código.  
* **phases:** Define a sequência de comandos a serem executados:  
  * install (instalação de dependências).  
  * pre\_build (comandos antes da compilação).  
  * build (comandos de compilação reais).  
  * post\_build (comandos de finalização, como empacotamento).  
* **artifacts:** Especifica quais arquivos devem ser empacotados e persistidos no S3 como a **saída** da fase de *build* (os *artifacts* de implantação).  
* **cache:** Define arquivos (geralmente dependências) que devem ser armazenados em um *bucket* S3 para acelerar *builds* futuros.

---

## **5\. AWS CodeDeploy (Implantação Automatizada)**

O CodeDeploy automatiza a implantação de revisões de aplicações em diferentes plataformas, permitindo *rollbacks* e controle de tráfego.

### **Plataformas de *Deployment***

| Plataforma | Mecanismo de Implantação |
| :---- | :---- |
| **EC2 / On-Premises** | Requer o **CodeDeploy Agent** instalado na instância/servidor. Usa o arquivo **appspec.yml** para definir o ciclo de vida do *deployment* (*hooks*). |
| **AWS Lambda** | Automatiza a **mudança de tráfego** usando **Lambda Aliases** e estratégias *Canary* ou *Linear*. |
| **ECS (Fargate/EC2)** | Suporta apenas *Blue/Green Deployments* para alternar entre *Target Groups* no Load Balancer. |

### **Estratégias de Implantação (EC2/On-Premises)**

1. **In-Place Deployment:** A versão antiga (v1) é atualizada diretamente nas instâncias existentes. Causa *downtime* (interrupção), dependendo da configuração.  
   * **Configurações de Velocidade:** **AllAtOnce** (mais rápido, mais *downtime*), **HalfAtATime**, **OneAtATime** (mais lento, menor impacto na disponibilidade).  
2. **Blue/Green Deployment:**  
   * **Blue (Azul):** O ambiente de produção atual (v1).  
   * **Green (Verde):** Um novo conjunto de instâncias (ou *Task Definitions* no ECS) com a nova versão (v2) é provisionado paralelamente.  
   * O Load Balancer transfere o tráfego do Blue para o Green (sem interrupção).  
   * Permite testar a v2 ao vivo antes de desativar a v1.

### **Estratégias de Tráfego (Lambda e ECS)**

* **Linear:** Aumenta o tráfego gradualmente (ex: 10% a cada 3 minutos) até 100%.  
* **Canary:** Move uma pequena porcentagem (ex: 10%) para a nova versão por um curto período (ex: 5 minutos). Se não houver alarmes, muda o tráfego restante para 100%.  
* **AllAtOnce:** Muda imediatamente de v1 para v2.

### **Rollbacks**

* O rollback ocorre **automaticamente** (em caso de falha de *deployment* ou alarme do CloudWatch) ou **manualmente**.  
* **Mecanismo:** O CodeDeploy não "restaura" o estado, ele executa um **novo *deployment*** da **última revisão conhecida como funcional** (Last Known Good Revision).

---

## **6\. AWS CodeArtifact (Gerenciamento de Artefatos)**

O CodeArtifact é um sistema de gerenciamento de artefatos seguro, escalável e totalmente gerenciado, essencial para lidar com dependências de código (*Code Dependencies*) em projetos de software.

### **6.1 Estrutura e Conceitos**

* **Domínios (Domains):** Um domínio é um contêiner lógico para um conjunto de repositórios.  
* **Repositórios (Repositories):** Armazenam os artefatos de código, geralmente de um tipo específico (ex: Python, Java, NPM).  
* **Integração:** O CodeArtifact se integra diretamente com ferramentas comuns de gerenciamento de dependências, como **Maven**, **Gradle**, **npm**, **pip** e **NuGet**.  
* **Localização:** Todos os artefatos residem de forma segura dentro da sua **VPC (AWS Cloud)**.

### **6.2 Funcionalidades Principais**

1. **Proxy para Repositórios Públicos:** O CodeArtifact atua como um *proxy* para repositórios públicos (ex: npm registry, PyPI).  
   * Desenvolvedores e CodeBuild buscam as dependências no **CodeArtifact**, que, por sua vez, busca no repositório público.  
   * **Benefícios:**  
     * **Segurança de Rede:** Isola os desenvolvedores do acesso direto a repositórios externos.  
     * **Cache de Dependências:** As dependências buscadas são **armazenadas em *cache*** no CodeArtifact. Isso garante que seu código possa ser construído futuramente, mesmo que uma dependência seja removida do repositório público.  
2. **Publicação de Artefatos Próprios:** Você pode publicar e aprovar seus próprios artefatos (pacotes internos) nos repositórios do CodeArtifact, centralizando todas as dependências em um só lugar.

### **6.3 Automação e Segurança**

* **EventBridge:** Eventos no CodeArtifact (como a criação, modificação ou exclusão de um pacote) podem emitir eventos para o **EventBridge**. O EventBridge pode então acionar serviços downstream, como o **CodePipeline** ou **AWS Lambda**, para iniciar reconstruções automatizadas (ex: reconstruir a aplicação com uma dependência de segurança atualizada).  
* **Acesso entre Contas (Cross-Account Access):** Para permitir que usuários e *roles* de outra conta AWS acessem um repositório CodeArtifact, é necessário configurar uma **Resource Policy** (Política de Recurso) no próprio repositório.

---

## **7\. Amazon CodeGuru (Revisão e Otimização de Código com ML)**

O CodeGuru é um serviço baseado em Machine Learning (Aprendizado de Máquina) que ajuda a melhorar a qualidade, a segurança e a performance do código. É dividido em duas ferramentas principais:

### **7.1 CodeGuru Reviewer (Revisão de Código Automatizada)**

* **Função:** Realiza análise estática de código (revisão de código) em repositórios (GitHub, Bitbucket, CodeCommit) a cada *commit*.  
* **Detecção:** Identifica **vulnerabilidades de segurança** (seguindo padrões como o OWASP Top 10), *bugs*, *resource leaks* e desvios de melhores práticas de codificação.  
* **Aprendizado:** Utiliza ML, aprendendo com milhares de repositórios de código aberto e da Amazon.com, para fornecer recomendações acionáveis na linha de código exata.  
* **Linguagens Suportadas:** Atualmente suporta **Java** e **Python**.

### **7.2 CodeGuru Profiler (Análise de Performance em Runtime)**

* **Função:** Analisa o comportamento e o desempenho da aplicação em **tempo de execução** (produção ou pré-produção).  
* **Otimização:** Identifica as "linhas de código caras" (*expensive lines of code*) que consomem excessivamente a capacidade da CPU ou da memória (Heap Summaries).  
* **Benefícios:**  
  * Melhora o desempenho da aplicação.  
  * Reduz os custos de computação (ao diminuir a utilização da CPU).  
  * Pode detectar anomalias no comportamento da aplicação.  
* **Execução:** Funciona através de um **agente** com sobrecarga mínima (*minimal overhead*) instalado na aplicação, que envia relatórios para o serviço. Suporta aplicações rodando na AWS ou *on-premises*.

### **7.3 Configurações do Agente Profiler (Conceitos Chave)**

O agente do CodeGuru Profiler pode ser ajustado:

* **MaxStackDepth:** Define a profundidade máxima da chamada de métodos que será perfilada.  
* **Sampling Interval:** O intervalo em milissegundos com que as amostras de perfilamento são coletadas. Um valor mais baixo resulta em uma taxa de amostragem mais alta.  
* **Minimum time for reporting / Reporting interval:** Define a frequência mínima com que o agente envia os relatórios de perfilamento para o CodeGuru.
