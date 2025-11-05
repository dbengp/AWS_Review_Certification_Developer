# **ElastiCache: Redis e Memcached**

## 1. Conceito e Objetivo Principal
* O **Amazon ElastiCache** é um serviço gerenciado de *caching* em memória de alto desempenho e baixa latência. Seu principal objetivo é **descarregar a carga de leitura de bancos de dados persistentes (RDS/Aurora)** e fornecer um armazenamento de dados ultrarrápido para sessões e dados temporários.

### **2\. Visão Geral, Benefícios e Arquitetura**

#### **A. Benefícios Principais**

* **Redução de Carga:** Alivia a pressão em bancos de dados (DB) para cargas de trabalho intensivas em leitura.  
* **Latência:** Oferece latência de resposta na ordem de milissegundos.  
* **Aplicação Sem Estado:** Armazena dados de sessão (estado do usuário) no cache, permitindo que as instâncias da aplicação sejam *stateless* e possam ser dimensionadas ou substituídas sem perda de dados do usuário.  
* **Gerenciamento AWS:** A AWS cuida do *patching* do SO, otimização, monitoramento e recuperação de falhas.

#### **B. Arquitetura Básica (Cache-Aside / Lazy Loading)**

1. **Cache Hit (Acerto):** A aplicação consulta o ElastiCache. Se os dados existirem, a resposta é entregue diretamente, economizando a viagem ao DB.  
2. **Cache Miss (Falta):** Se os dados não existirem, a aplicação lê os dados do DB (RDS), e então **escreve esses dados de volta no cache** para futuras solicitações.  
   * **Impacto:** O uso do cache exige **alterações no código** da aplicação.

---

### **3\. Mecanismos Suportados e Diferenças (Certificação)**

O ElastiCache suporta dois motores, cada um com características distintas de HA (Alta Disponibilidade) e uso:

| Característica | Redis | Memcached |
| :---- | :---- | :---- |
| **Tipo de Uso** | Estruturas de dados complexas, ranking, filas. | *Caching* simples de objetos e resultados. |
| **Alta Disponibilidade** | **Suporta Multi-AZ com Failover Automático.** | **Não** suporta failover automático (em modo não *serverless*). |
| **Durabilidade** | Suporta **Persistência de Dados** (AOF / RDB) e Backup/Restauração. | **Não** suporta persistência ou backups (em modo não *serverless*). Se o nó falhar, o cache é perdido. |
| **Escalabilidade** | Suporta **Réplicas de Leitura** (para escalar leituras) e **Clustering** (*sharding*). | Suporta **Sharding** (múltiplos nós particionados) e arquitetura *multi-thread*. |
| **Recomendado para** | Sessões de usuário, tabelas de classificação (usando *Sorted Sets*), e dados que exigem HA. | Descarregar resultados de consultas e *caching* de objetos simples. |

---

### **4\. Estratégias de Caching (Padrões de Design)**

O exame espera que o candidato saiba ler e entender a lógica por trás de dois padrões principais:

#### **A. Cache-Aside / Lazy Loading (Carregamento Preguiçoso)**

* **Lógica:** A aplicação verifica o cache na leitura. Em caso de *Cache Miss*, ela busca no DB e **popula o cache**.  
* **Vantagem:** Apenas os dados **solicitados** são armazenados em cache, tornando-o eficiente.  
* **Desvantagem (Penalidade):** Em caso de *Cache Miss*, há uma **penalidade de latência de leitura** (três chamadas de rede: cache miss, leitura no DB, escrita no cache).  
* **Consistência:** Os dados podem ficar **obsoletos** se forem alterados no DB e o cache não for invalidado (Consistência Eventual).  
* **Código (Leitura):** A função get\_user(id) primeiro chama cache.get(id), e só se for None, chama db.query().

#### **B. Write-Through (Escrita Através)**

* **Lógica:** A aplicação escreve dados **simultaneamente no DB e no Cache**.  
* **Vantagem:** Os dados no cache **nunca são obsoletos** após uma gravação.  
* **Desvantagem (Penalidade):** Há uma **penalidade de latência de gravação** (duas chamadas de rede: escrita no DB e escrita no cache). Isso é aceitável, pois usuários esperam que operações de escrita sejam mais lentas.  
* **Consistência:** Garante maior estabilidade do cache, mas o cache pode ter dados **não lidos** se houver muitas gravações e poucas leituras.  
* **Código (Escrita):** A função save\_user(record) primeiro chama db.query() e, em seguida, chama cache.set().

---

### **5\. Gerenciamento de Cache e Invalidação**

A "invalidação de cache e nomeação de coisas" é considerada um dos problemas mais difíceis da Ciência da Computação.

#### **A. Despejo de Cache (Cache Eviction)**

Quando o cache fica cheio, itens precisam ser removidos.

* **LRU (Least Recently Used):** Padrão comum, remove os itens que foram usados há mais tempo.  
* **Exclusão Explícita:** O aplicativo remove o item do cache (geralmente após uma atualização no DB).  
* **TTL (Time-To-Live):** Define um tempo de vida para o item.  
  * **Uso:** Mesmo um TTL de poucos segundos pode ser muito eficaz. Ajuda a equilibrar a atualização dos dados e a conservação de memória.

#### **B. Implicações Operacionais**

* **Aquecimento (Cache Warming):** Após uma falha ou limpeza do cache, as primeiras solicitações resultarão em *Cache Misses* e terão alta latência até que o cache seja "aquecido" (preenchido com dados).

---

### **6\. ElastiCache Serverless e Implicações para a Certificação ➕**

#### **A. Amazon ElastiCache Serverless**

É o modo mais recente e mais fácil de operar.

* **Gerenciamento:** A AWS gerencia o *sharding*, o dimensionamento (*scaling*) e o *placement* dos nós.  
* **Capacidade:** O serviço escala automaticamente a capacidade de *caching* conforme a demanda da aplicação.  
* **Foco no Custo:** O usuário paga pela memória e capacidade de processamento (ECP) consumidas.

#### **B. Integração e Segurança**

* **VPC:** O ElastiCache é implantado em uma **VPC (Virtual Private Cloud)** e só pode ser acessado por instâncias EC2, ECS Tasks ou Lambdas **dentro dessa mesma VPC** ou via *Peering*.  
* **Segurança:** O acesso é controlado via **Security Groups**, que atuam como um *firewall* virtual de nível de instância.  
* **IAM:** O acesso às APIs de gerenciamento do ElastiCache é controlado via IAM.
