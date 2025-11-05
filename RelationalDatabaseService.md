# **Relational Database Service (RDS) da AWS**

## 1. Conceito e Objetivo Principal
* É um serviço gerenciado que facilita a configuração, operação e escalabilidade de bancos de dados relacionais (baseados em SQL) na nuvem.
* O RDS abstrai tarefas operacionais complexas, permitindo que o usuário se concentre na aplicação.
* O RDS é um serviço gerenciado compatível com diversos mecanismos de banco de dados, sendo a principal alternativa para não ter que provisionar e manter um banco de dados em uma instância EC2.

#### **A. Mecanismos Compatíveis**

É crucial lembrar os mecanismos suportados pelo RDS:

* **Proprietários da AWS:** Amazon Aurora (Estudado em profundidade).  
* **Open Source:** PostgreSQL, MySQL, MariaDB.  
* **Comerciais:** Oracle, Microsoft SQL Server.

#### **B. Tarefas Gerenciadas pela AWS**

A AWS automatiza diversas tarefas, permitindo que o usuário não precise fazer **SSH** nas instâncias subjacentes:

* Provisionamento e *Patching* do sistema operacional.  
* Backups Contínuos (*Point in Time Restore* para qualquer momento).  
* Configuração de Réplicas de Leitura e Multi-AZ.  
* Dimensionamento (*Scaling*), tanto vertical (tipo de instância) quanto horizontal (Réplicas).

#### **C. RDS Storage Auto Scaling**

Este recurso permite que o armazenamento do banco de dados aumente automaticamente em resposta ao crescimento da demanda, evitando a falta de espaço livre.

* **Ativação:** Requer a definição de um limite **máximo de armazenamento**.  
* **Condições de Aumento:** O armazenamento será aumentado automaticamente se:  
  1. O armazenamento livre for inferior a 10% do alocado, **E**  
  2. O baixo armazenamento durar mais de cinco minutos, **E**  
  3. Seis horas tiverem se passado desde a última modificação.  
* **Caso de Uso:** Ideal para aplicações com carga de trabalho **imprevisível**.

---

### **2\. Alta Disponibilidade e Escalabilidade (Read Replicas vs. Multi AZ)**

É o tópico mais importante do RDS para o exame, pois define a diferença entre escalabilidade de leitura e recuperação de desastres.

#### **A. Read Replicas (Réplicas de Leitura)**

Usadas para **Escalabilidade de Leitura** (SELECTs).

* **Número Máximo:** Até 15 réplicas de leitura por instância mestre.  
* **Localização:** Podem ser criadas na mesma AZ, em AZs diferentes ou em **Regiões Diferentes (Cross-Region)**.  
* **Replicação:** É **Assíncrona**, o que resulta em **Consistência Eventual**.  
* **Promoção:** Uma Réplica de Leitura pode ser **promovida** a sua própria instância de banco de dados principal.  
* **Custo de Rede:** A replicação entre AZs **dentro da mesma Região** é **gratuita**. A replicação **Cross-Region** incorre em custo de rede.  
* **Caso de Uso:** Descarregar cargas de trabalho pesadas de leitura, como *reporting* e análise, da instância principal.

#### **B. Multi AZ (Multiplas Zonas de Disponibilidade)**

Usado para **Recuperação de Desastres (Disaster Recovery)** e Alta Disponibilidade.

* **Replicação:** É **Síncrona** para uma instância *standby* dedicada. Garante que os dados na *standby* sejam idênticos ao *Master* em tempo real.  
* **Disponibilidade:** Proporciona um único nome DNS. Em caso de falha da instância *Master* (rede, instância, armazenamento ou falha de AZ), o **Failover é Automático** para a *standby*.  
* **Escalabilidade:** **Não** é usado para escalabilidade de leitura. A instância *standby* está indisponível para leituras/escritas (apenas em modo de espera).  
* **Transição de Single para Multi AZ:** É uma **operação sem *downtime*** (tempo de parada). O RDS tira um *snapshot* e restaura em uma nova *standby*, estabelecendo a sincronização.

---

### **3\. Segurança e Criptografia**

O acesso e a segurança do RDS são controlados por uma combinação de criptografia, IAM e Security Groups.

#### **A. Criptografia em Repouso (Data at Rest)**

* **KMS:** A criptografia é realizada usando **AWS KMS** e é definida **no momento do lançamento**.  
* **Regra de Criptografia:** Se a instância principal **não for** criptografada, as réplicas de leitura **não poderão** ser criptografadas.  
* **Criptografar DB Existente:** Para criptografar um DB não criptografado, é necessário **tirar um *snapshot*** e **restaurá-lo como um novo banco de dados criptografado**.

#### **B. Criptografia em Trânsito (Data in Transit)**

* Todos os bancos de dados RDS/Aurora suportam criptografia em trânsito por padrão (TLS/SSL). Os clientes devem utilizar os certificados de raiz TLS da AWS.

#### **C. Autenticação e Acesso**

* **Autenticação:** Pode ser feita usando o clássico **nome de usuário e senha** ou através de **funções do IAM**, o que é preferido para integração com instâncias EC2.  
* **Controle de Rede:** O acesso é controlado por **Security Groups**, permitindo ou bloqueando portas e IPs específicos.  
* **Auditoria:** É possível habilitar **Logs de Auditoria** e enviá-los para o **CloudWatch Logs** para retenção de longo prazo.

---

### **4\. Amazon RDS Proxy**

O RDS Proxy é um serviço totalmente gerenciado e *serverless* que atua como um *pool* de conexões entre as aplicações e a instância de banco de dados.

#### **A. Benefícios Principais**

1. **Agrupamento de Conexões (*Connection Pooling*):** Reduz o número de conexões abertas com a instância de banco de dados, diminuindo a tensão sobre os recursos (CPU/RAM) do DB.  
   * **Caso de Uso Central:** Essencial ao usar serviços como **AWS Lambda**, que podem criar e encerrar milhares de conexões rapidamente, sobrecarregando o banco de dados.  
2. **Redução do Tempo de Failover:** Reduz o tempo de *failover* em até **66%** durante uma troca de Multi-AZ, pois a aplicação só precisa se reconectar ao Proxy, que gerencia o *failover* do DB subjacente de forma transparente.  
3. **Reforço de Segurança:** Permite **reforçar a autenticação do IAM** para o banco de dados e gerenciar credenciais com segurança no **AWS Secrets Manager**.

#### **B. Características**

* **Acessibilidade:** O RDS Proxy **nunca é acessível publicamente**. É acessível apenas de dentro do VPC.  
* **Mecanismos Suportados:** MySQL, PostgreSQL, MariaDB, Microsoft SQL Server, e Aurora (MySQL/PostgreSQL).  
* **Mudança de Código:** Não requer alterações de código na aplicação; basta mudar o *endpoint* de conexão.

### **5\. Considerações Importantes**

#### **A. Caching para Máxima Escalabilidade (ElastiCache)**

Para descarregar tráfego de leitura **pesado** e alcançar a maior escalabilidade de leitura possível, a solução de arquitetura é usar uma camada de *caching* em memória (cache lado a lado) **na frente** do RDS.

* **Solução:** **Amazon ElastiCache** (com Redis ou Memcached).  
* **Estratégia:** O código da aplicação deve ser modificado para primeiro consultar o cache e, só se o dado não for encontrado (*cache miss*), buscar no RDS (padrão **Lazy Loading**).

#### **B. Grupos de Parâmetros e Grupos de Opções**

O RDS usa grupos para permitir customização de baixo nível sem acessar o sistema operacional:

* **DB Parameter Group:** Usado para ajustar parâmetros de configuração do mecanismo de banco de dados (ex: max\_connections, *timeouts*).  
* **DB Option Group:** Usado para habilitar recursos opcionais do DB (ex: integração com Active Directory, TDE para Oracle).

#### **C. Upgrades de Versão Principal (Major Version Upgrades)**

* **Risco:** Um *upgrade* de versão principal (ex: MySQL 5.7 para 8.0) é uma operação que requer **downtime** e pode introduzir **mudanças incompatíveis** no banco de dados ou no código da aplicação.  
* **Melhor Prática:** É essencial testar a compatibilidade em um ambiente de *staging* ou usar o serviço **Blue/Green Deployment** para minimizar o impacto.
