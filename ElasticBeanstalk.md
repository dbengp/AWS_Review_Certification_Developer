# **🛠️ Elastic Beanstalk**

## 1. Conceito e Objetivo Principal

O **Elastic Beanstalk (EB)** é um serviço de orquestração de implantação e infraestrutura **centrado no desenvolvedor**. Seu objetivo é abstrair a complexidade de provisionar e gerenciar a infraestrutura subjacente (EC2, ASG, ELB, RDS) para que o desenvolvedor possa focar apenas no código.

🔑 **Ponto Chave:** O serviço EB em si é gratuito; você paga pelos recursos AWS subjacentes que ele provisiona (EC2, ELB, RDS, etc.).

| Componente | Descrição |
| :---- | :---- |
| **Application (Aplicação)** | É o contêiner lógico para as versões de código, configurações e ambientes (ex: MyWebApp). |
| **Application Version (Versão da Aplicação)** | Uma iteração específica do seu código (upload de um arquivo .zip ou Docker image). |
| **Environment (Ambiente)** | É a coleção de recursos da AWS (EC2, ASG, ELB) que executa uma **versão** específica da aplicação. |
| **Configuration (Configuração)** | As configurações de infraestrutura (tipo de instância, tamanho do ASG, variáveis de ambiente, etc.). |

### **2\. Tiers de Ambiente (Camadas)**

Existem dois modelos arquiteturais que o EB pode implantar:

| Tier | Arquitetura | Caso de Uso |
| :---- | :---- | :---- |
| **Web Server Environment** | **Tradicional:** Possui um **Load Balancer** (ELB/ALB) distribuindo o tráfego externo para o **Auto Scaling Group (ASG)** de instâncias EC2. | Hospedagem de API, sites e aplicações web voltadas ao público. |
| **Worker Environment** | **Processamento Assíncrono:** As instâncias EC2 atuam como **trabalhadores** que puxam mensagens de uma **SQS Queue** (fila) provisionada automaticamente. | Processamento em lote, tarefas demoradas, processamento de pagamento fora de banda. O dimensionamento é baseado na contagem de mensagens na SQS. |

### **3\. Modos de Capacidade e Alta Disponibilidade**

O EB permite dois modos de configuração de capacidade:

1. **Single Instance (Instância Única):** Ótimo para **Desenvolvimento e Teste**. Provisiona apenas uma instância EC2 (com um IP Elástico) e **não** possui um Load Balancer ou ASG.  
2. **High Availability with Load Balancer (Alta Disponibilidade):** **Recomendado para Produção.** Provisiona um ELB/ALB, ASG com múltiplas instâncias em múltiplas AZs.

### **4\. Estratégias de Implantação**

O exame testa cenários que exigem tempo de inatividade zero ou rollback rápido.

| Método de Implantação | Tempo de Inatividade (Downtime) | Custo Adicional | Rollback | Caso de Uso |
| :---- | :---- | :---- | :---- | :---- |
| **All at Once** | **SIM** (Todas as instâncias param para atualizar). | Não | Rápido (re-implantar V1). | Ambientes de **Desenvolvimento** ou onde o *downtime* é aceitável. **Mais Rápido.** |
| **Rolling** | **SIM** (Capacidade Reduzida). | Não | Lento (re-implantar em *batches*). | Atualiza um **batch** de instâncias por vez. A aplicação roda abaixo da capacidade total. |
| **Rolling with Additional Batch** | **NÃO** (Sempre roda na capacidade total). | **SIM** (Instâncias adicionais temporárias). | Lento. | Adiciona instâncias extras para atualizar em *batches*, mantendo a capacidade. Bom para **Produção** com custo pequeno. |
| **Immutable (Imutável)** | **NÃO** (Zero Downtime). | **SIM** (Dobra a capacidade temporariamente). | **RÁPIDO** (apenas encerrar o novo ASG). | Lança um **ASG temporário** totalmente novo. Se saudável, substitui o antigo. Ótima escolha para **Produção** com *rollback* rápido. |
| **Traffic Splitting (Canary Testing)** | **NÃO** (Zero Downtime). | **SIM** (ASG temporário). | **AUTOMÁTICO e RÁPIDO** (simplesmente remove o tráfego do novo ASG). | Envia uma pequena % de tráfego (ex: 10%) para a nova versão por um tempo configurável. Melhor para **Canary Testing** automatizado. |
| **Blue/Green** | **NÃO** (Zero Downtime). | **SIM** (Dois ambientes ativos). | Rápido (trocar o DNS de volta). | Envolve criar um **novo Ambiente EB** ("Green"). Requer uma **troca de URL** manual ou via Route 53\. **Não é nativo do EB, mas uma prática recomendada.** |

### **5\. Personalização e Extensões (Arquivos .ebextensions)**

Como desenvolvedor, você pode personalizar o ambiente e a infraestrutura usando arquivos de configuração:

* **Localização:** Os arquivos de configuração devem estar no diretório **.ebextensions/** na raiz do seu pacote .zip.  
* **Formato:** Devem ser arquivos .yaml ou .json com a extensão **.config** (ex: logging.config).  
* **Finalidade:**  
  * **option\_settings:** Usado para definir variáveis de ambiente (DB\_URL, DB\_USER) e modificar configurações de recursos gerenciados (EC2, ASG, ELB).  
  * **resources:** Usado para provisionar recursos **Adicionais** da AWS que não são gerenciados nativamente (ex: um banco de dados RDS dedicado, um ElastiCache).  
* **Ciclo de Vida:** Os recursos criados via .ebextensions **serão excluídos** se o ambiente for encerrado (a menos que explicitamente configurados para retenção).

### **6\. Gerenciamento e Ciclo de Vida**

* **Pacotes de Código-Fonte:** O arquivo .zip da sua aplicação é carregado no **Amazon S3** (em um bucket gerenciado pelo EB) e registrado no Beanstalk como uma **Application Version**.  
* **Limitação de Versões:** O EB tem um limite de **1.000 versões** de aplicativos por conta.  
* **Política de Ciclo de Vida:** Você deve usar a **Política de Ciclo de Vida de Aplicações** para remover automaticamente versões antigas (baseado em contagem máxima ou idade) e evitar atingir o limite.

### **7\. Elastic Beanstalk e CloudFormation (Infraestrutura como Código)**

O Elastic Beanstalk utiliza o **AWS CloudFormation** nos bastidores para provisionar e gerenciar todos os recursos da infraestrutura (EC2, ASG, ELB, RDS).

* **Abstração:** O desenvolvedor interage apenas com a interface do EB, e o EB traduz isso em modelos CloudFormation.  
* **Transparência:** Para cada ambiente EB, uma **Pilha (Stack) CloudFormation** é criada para gerenciar os recursos.  
* **Customização Avançada:** O ponto de poder para o desenvolvedor é o uso dos arquivos **.ebextensions**. Dentro desses arquivos, você pode injetar trechos de CloudFormation (na seção resources) para provisionar **qualquer outro serviço AWS** que não seja nativo do EB (ex: DynamoDB, S3 Bucket, ElastiCache).

🔑 **Foco na Prova:** Embora o EB gerencie a infraestrutura, o desenvolvedor pode expandir a pilha usando **.ebextensions** para adicionar recursos personalizados via sintaxe CloudFormation.

### **8\. Gerenciamento de Ambientes**

#### **8.1. Clonagem de Ambiente (Clone Environment)**

* **Propósito:** Criar um novo ambiente com **exatamente a mesma configuração** (tipo de ELB, ASG, variáveis de ambiente, etc.) do ambiente original.  
* **Uso:** Ideal para criar um ambiente de *staging* ou teste com a mesma infraestrutura de produção.  
* **Ressalva:** Se houver um banco de dados RDS provisionado **dentro** do ambiente, a **configuração** do RDS é preservada, mas os **dados não são copiados** (o novo RDS inicia vazio).

#### **8.2. Troca de URL (Swap Environment URLs)**

* Uma funcionalidade de gerenciamento de tráfego que facilita a estratégia **Blue/Green** (manual).  
* Permite trocar os CNAMEs (URLs de ambiente) entre dois ambientes EB rodando versões diferentes da aplicação, redirecionando o tráfego do "Blue" (antigo) para o "Green" (novo) instantaneamente.

### **9\. Cenários de Migração (Decoupling e Mudança de Load Balancer)**

O exame adora testar como você lida com componentes quando eles precisam de ciclos de vida separados do ambiente EB.

#### **9.1. Migração de Tipo de Load Balancer (ELB)**

* **Restrição:** **Não é possível** alterar o **tipo** de Load Balancer (ex: Classic para Application) em um ambiente EB **existente**.  
* **Solução de Migração (Manual):**  
  1. Crie um **novo ambiente EB** com a mesma configuração, mas **especificando o novo tipo** de Load Balancer (ALB ou NLB).  
  2. Implante a aplicação no novo ambiente.  
  3. Transfira o tráfego: Use **Troca de CNAME** (se aplicável) ou **atualize o DNS** (Route 53\) para apontar para o novo ambiente.  
  4. Encerre o ambiente antigo.  
* **Por que não usar Clone?** O recurso de **Clone** copia o tipo de Load Balancer, o que inviabiliza o objetivo da migração.

#### **9.2. Desacoplamento do Banco de Dados RDS (Melhor Prática de Produção)**

* **Problema:** No desenvolvimento, ter o RDS acoplado é conveniente. Em **produção**, se o RDS for gerenciado pelo EB, **ele será excluído** quando o ambiente EB for encerrado, resultando em perda de dados.  
* **Melhor Prática (RDS Externo):** O RDS de produção deve ser provisionado **manualmente e fora** do ambiente EB, sendo referenciado pelo aplicativo via **variáveis de ambiente** (DB\_URL, DB\_USER).  
* **Processo de Desacoplamento de um RDS Existente (Migração):**  
  1. Crie um **Snapshot** do RDS (segurança).  
  2. Vá ao console RDS e ative a **Proteção Contra Exclusão** no banco de dados.  
  3. Crie um **novo ambiente EB sem RDS** e aponte-o para o RDS existente (usando variáveis de ambiente).  
  4. Faça a **Troca de CNAME** (ou DNS) para o novo ambiente.  
  5. Encerre o ambiente antigo (EB tentará excluir a pilha CFN, mas o RDS será protegido).  
  6. **Ação Manual:** Exclua a pilha CloudFormation **manualmente** (ela estará no estado DELETE\_FAILED) para limpar os recursos restantes, mantendo o RDS intacto.
