# **🚀 Amazon DynamoDB**

## 1. Conceito e Objetivo Principal

O Amazon DynamoDB é um **serviço de banco de dados NoSQL totalmente gerenciado** da AWS, projetado para oferecer desempenho rápido e consistente em qualquer escala. É um serviço **serverless**, o que significa que não é necessário gerenciar a infraestrutura subjacente (servidores).

### **1\. 🌐 DynamoDB vs. Bancos de Dados Relacionais (RDBMS)**

| Característica | RDBMS (Ex: RDS) | DynamoDB (NoSQL) |
| :---- | :---- | :---- |
| **Tipo** | Relacional (SQL) | Não Relacional (NoSQL/Não Apenas SQL) |
| **Escalabilidade** | Predominantemente **Vertical** (melhor CPU/RAM/Disco) e **Escala de Leitura Horizontal** (Réplicas de Leitura). | Predominantemente **Horizontal** e totalmente **Distribuída** (escala para milhões de requisições/segundo). |
| **Modelo de Dados** | Schema Fixo, Tabelas, JOINs, Agregações. | Schema Flexível (sem esquemas rígidos), Tabelas, Itens (linhas), Atributos (colunas). |
| **Performance** | Boa, mas limitada pela escala vertical ou pelo limite de réplicas de leitura. | **Rápida e Consistente** com latência muito baixa, escalando para cargas de trabalho massivas. |
| **Operações Complexas** | Suporta JOINs, Agregações (SUM, AVG, etc.). | Não suporta (ou tem suporte muito limitado) para JOINs. Agregações são geralmente feitas na camada de aplicação. |

---

### **2\. 🧱 Conceitos Básicos e Modelagem de Dados**

DynamoDB organiza os dados em **Tabelas** que contêm **Itens** (linhas), e cada item tem **Atributos** (colunas).

* **Itens e Atributos:**  
  * Um Item pode ter até **400 KB** de dados.  
  * Atributos podem ser **aninhados** (usando tipos **List** e **Map**) e não precisam ser definidos no momento da criação da tabela. Um atributo pode estar ausente (NULL) em um item, o que confere flexibilidade de esquema.  
* **Tipos de Dados Suportados:**  
  * **Escalares:** String, Number, Binary, Boolean, Null.  
  * **Documentos:** List, Map (para aninhamento).  
  * **Sets:** String Set, Number Set, Binary Set.

#### **Chave Primária (Primary Key) \- Foco do Exame\! 🔑**

A chave primária é fundamental para a distribuição dos dados e deve ser escolhida antes da criação da tabela. Existem duas opções:

1. **Chave de Partição (Partition Key)** \- Estratégia "Hash":  
   * A Chave de Partição deve ser **única** para cada item.  
   * O valor da chave é passado por um algoritmo de *hashing* para determinar em qual **Partição** o item será armazenado.  
   * **Escolha:** Deve ter a **maior cardinalidade** (grande diversidade de valores) para garantir uma **distribuição de dados uniforme** e evitar "partições quentes" (*hot partitions*). Exemplo: movie\_id é melhor que movie\_language.  
2. **Chave de Partição \+ Chave de Classificação (Sort Key)** \- Estratégia "Hash \+ Range":  
   * A **combinação** da Chave de Partição e da Chave de Classificação deve ser **única** para cada item.  
   * Os dados são **agrupados** pela Chave de Partição e **ordenados** pela Chave de Classificação dentro da partição.  
   * Isso permite operações de consulta baseadas em **intervalos** (ex: *greater than, begins with, between*) na Chave de Classificação.

---

### **3\. ⚖️ Capacidade de Leitura e Escrita (RCU & WCU)**

DynamoDB oferece dois modos para gerenciar o throughput (taxa de transferência). É possível alternar entre os modos uma vez a cada 24 horas.

#### **A. Modo Provisionado (Provisioned Mode) \- RCU/WCU**

Você especifica a capacidade de leitura e escrita por segundo antecipadamente, em unidades. Você paga pelo que provisiona.

| Unidade | Nome | Definição |
| :---- | :---- | :---- |
| **WCU** | Write Capacity Unit | Uma WCU \= **1 gravação por segundo** para um item de **até 1 KB**. |
| **RCU** | Read Capacity Unit | Uma RCU \= **1 leitura fortemente consistente por segundo** (ou **2 leituras eventualmente consistentes por segundo**) para um item de **até 4 KB**. |

* **Arredondamento:** Para WCU e RCU, o tamanho do item é **sempre arredondado para cima** para o próximo múltiplo da unidade base (1 KB para WCU, 4 KB para RCU). Ex: Um item de 4.5 KB para WCU é tratado como 5 KB (5 WCUs). Um item de 6 KB para RCU é tratado como 8 KB (2 unidades de 4 KB).  
* **Capacidade de *Burst***: Capacidade temporária disponível para lidar com picos de tráfego que excedem o provisionado.  
* **Throttling:** Se o consumo de RCU/WCU exceder o provisionado (e o *burst* for esgotado), o DynamoDB retorna uma exceção ProvisionedThroughputExceededException.  
  * **Solução:** Implementar a **Estratégia de *Exponential Backoff*** (re-tentativa com tempos de espera crescentes) no lado do cliente ou aumentar as WCUs/RCUs.  
* **Auto Scaling:** Permite ajustar as RCUs e WCUs automaticamente dentro de limites definidos para atender à demanda.

#### **B. Modo On-Demand (On-Demand Mode) \- RRU/WRU**

A capacidade de leitura e escrita escala automaticamente para cima e para baixo com base na sua carga de trabalho.

* **Vantagens:** Não requer planejamento de capacidade; sem throttling. Ideal para **cargas de trabalho imprevisíveis** ou desconhecidas.  
* **Desvantagens:** É **mais caro** (aprox. 2.5x) que o Modo Provisionado.  
* **Unidades:** Cobrança baseada em **Unidades de Requisição de Leitura (RRU)** e **Unidades de Requisição de Escrita (WRU)**.

#### **C. Consistência de Leitura**

| Modo | Definição | Custo RCU | Latência | Uso |
| :---- | :---- | :---- | :---- | :---- |
| **Eventualmente Consistente** | **Padrão.** Pode retornar dados **stale** (antigos) logo após uma escrita, pois a replicação entre as partições pode demorar (em torno de 100ms). | **2 Leituras/RCU** | Baixa | A maioria dos casos de uso onde a consistência imediata não é crítica. |
| **Fortemente Consistente** | **Não é o padrão.** Garante que o item lido é a versão mais recente após uma escrita bem-sucedida. | **1 Leitura/RCU** (o dobro do Eventually Consistent) | Ligeiramente maior | Casos críticos onde a leitura deve refletir a escrita mais recente (Ex: transações financeiras). Requer a configuração de ConsistentRead como True na API. |

---

### **4\. ⚙️ Operações de API**

As interações com o DynamoDB são feitas através de APIs que podem ser executadas usando o AWS SDK, CLI, ou o **PartiQL** (uma linguagem de consulta compatível com SQL).

#### **A. Operações de Escrita**

| API | Descrição | Observação |
| :---- | :---- | :---- |
| PutItem | Cria um novo item ou **substitui totalmente** um item existente com a mesma Primary Key. | Para substituição total. |
| UpdateItem | **Edita** atributos existentes em um item ou adiciona um novo item se não existir. | Para edição parcial. Suporta Contadores Atômicos. |
| DeleteItem | Deleta um item individual. |  |
| **Escritas Condicionais** | Aceita uma operação de escrita/atualização/exclusão somente se uma **condição** for atendida (ex: PutItem se o atributo version for X). | Ajuda a gerenciar acessos concorrentes. |
| BatchWriteItem | Permite até **25** operações de PutItem e/ou DeleteItem em uma única chamada. | Não suporta UpdateItem. Retorna UnprocessedItems em caso de falha (geralmente por falta de WCU). |

#### **B. Operações de Leitura**

| API | Chaves Utilizadas | Escopo | Consumo de RCU |
| :---- | :---- | :---- | :---- |
| GetItem | **Chave Primária Completa** (Partition Key e opcionalmente Sort Key). | Lê **um único item**. | Muito eficiente. |
| Query | **Chave de Partição** (operador de igualdade) e **opcionalmente Chave de Classificação** (operadores de intervalo/comparação). | Lê **vários itens** que compartilham a mesma Partition Key. | Eficiente, focado em uma partição. |
| Scan | Nenhuma. | Lê **toda a tabela**. | **Muito Ineficiente** e consome muitas RCUs. Evitar em produção para operações normais. |
| BatchGetItem | Lista de Primary Keys. | Retorna **até 100 itens** de uma ou mais tabelas. | Permite paralelismo para minimizar latência. Retorna UnprocessedKeys em caso de falha na leitura. |

* **ProjectionExpression:** Usado em operações de leitura (GetItem, Query, Scan) para solicitar apenas **atributos específicos**, reduzindo a carga de rede e o processamento de dados desnecessários.  
* **FilterExpression:** Usado em operações de Query e Scan para **filtrar os resultados** **depois** que a consulta/leitura foi realizada, mas **antes** de retornar os dados. **Importante:** O filtro é aplicado após o consumo de RCU/WRU (o RCU é consumido pela leitura de *todos* os itens, independentemente do filtro).  
* **Paginação:** Todas as operações que podem retornar mais de 1 MB de dados (ou o limite especificado) exigem paginação para recuperar o conjunto completo de resultados.  
* **Parallel Scan:** Usa múltiplos *workers* para escanear a tabela em segmentos ao mesmo tempo, aumentando a taxa de transferência (throughput) à custa de um alto consumo de RCU.  
* **DeleteTable:** É a forma mais rápida de apagar **todos** os itens de uma tabela.

---

### **5\. 🧑‍💻 PartiQL**

PartiQL é uma linguagem de consulta **compatível com SQL** que pode ser usada para interagir com o DynamoDB.

* **Capacidade:** Suporta operações SELECT, INSERT, UPDATE e DELETE.  
* **Limitação:** **Não suporta JOINs**, mantendo o modelo de dados NoSQL do DynamoDB.  
* **Propósito:** Oferece uma sintaxe familiar para desenvolvedores que estão acostumados com SQL, sem adicionar novas funcionalidades não nativas ao DynamoDB.

---

### **6\. ✍️ Escritas Condicionais (Conditional Writes)**

Escritas condicionais permitem que você especifique uma **ConditionExpression** para as operações de escrita (PutItem, UpdateItem, DeleteItem e BatchWriteItem). A operação só será executada se a condição for avaliada como **verdadeira**.

| Operação | Finalidade | Exemplo de Condição |
| :---- | :---- | :---- |
| **Escrita Condicional** | Controla se uma operação de escrita (modificação) deve ser aplicada a um item. | Price \> :limit |
| **Expressão de Filtro** | Reduz o conjunto de resultados de uma operação de **leitura** (Query ou Scan). | Category \= 'Books' |

#### **Principais Condições**

* **attribute\_exists / attribute\_not\_exists**: Verifica se um atributo existe ou não.  
  * **Caso de Uso para Prevenção de Sobrescrita**: Usar attribute\_not\_exists no **Partition Key** (e no Sort Key, se aplicável) garante que uma operação PutItem não sobrescreva um item existente, funcionando como um *create-if-not-exists*.  
* **attribute\_type**: Checa o tipo de dado do atributo.  
* **contains / begins\_with**: Para comparação e verificação de strings.  
* **IN**: Verifica se o valor de um atributo está em uma lista de valores.  
* **BETWEEN :low AND :high**: Verifica se um valor numérico está em um determinado intervalo.  
* **size**: Retorna o tamanho de um atributo (útil para verificar o comprimento de uma string).

---

### **7\. 🗂️ Índices Secundários (Indexes)**

Os índices permitem que você consulte dados na sua tabela usando chaves diferentes da chave primária (Partition Key \+ Sort Key) principal.

#### **➡️ Índice Secundário Local (Local Secondary Index \- LSI)**

| Característica | Descrição |
| :---- | :---- |
| **Chave de Partição (Hash Key)** | **MESMA** que a da tabela base. |
| **Chave de Classificação (Sort Key)** | **DIFERENTE** da tabela base (sort key alternativo). |
| **Definição** | Deve ser definido **NA CRIAÇÃO DA TABELA** (não pode ser adicionado/removido posteriormente). |
| **Capacidade** | **Compartilha** as Unidades de Capacidade de Leitura e Escrita (**RCU & WCU**) da tabela base. |
| **Limite** | Máximo de **5 LSIs** por tabela. |

#### **🌐 Índice Secundário Global (Global Secondary Index \- GSI)**

| Característica | Descrição |
| :---- | :---- |
| **Chave Primária** | **DIFERENTE** da tabela base (tanto Partition Key quanto, opcionalmente, Sort Key). Permite novos padrões de consulta. |
| **Definição** | Pode ser adicionado, modificado ou removido **APÓS A CRIAÇÃO DA TABELA**. |
| **Capacidade** | Deve ter sua própria capacidade de RCU & WCU **PROVISIONADA SEPARADAMENTE**. |
| **Throttling (Estrangulamento)** | **Ponto Crítico de Exame**: Se as escritas forem estranguladas no **GSI**, a operação de escrita na **Tabela Base também será estrangulada**, mesmo que a tabela base tenha WCUs suficientes. Isso ocorre porque o GSI é atualizado de forma assíncrona pela escrita na tabela principal. |
| **Projeções** | Permite escolher quais atributos da tabela base serão copiados para o índice (**Projeção**). |

---

### **8\. 🔒 Bloqueio Otimista (Optimistic Locking)**

Bloqueio Otimista é um padrão implementado usando **Escritas Condicionais** para garantir que um item não tenha sido modificado por outro cliente entre o momento em que foi lido e o momento em que se tenta atualizá-lo ou excluí-lo.

* **Mecanismo**: Um atributo de **número de versão** é adicionado ao item.  
* **Processo**: A operação de UpdateItem ou DeleteItem inclui uma **ConditionExpression** que verifica se o número de versão atual no DynamoDB é o mesmo que o número de versão lido pelo cliente. Se a condição for atendida, a atualização prossegue e o número de versão é **incrementado**. Se a condição falhar, o cliente recebe um erro, indicando que o item foi alterado por outro processo, e deve tentar a operação novamente (lendo o novo item primeiro).

---

### **9\. 🚀 DynamoDB Accelerator (DAX)**

O DAX é um **cache em memória totalmente gerenciado e altamente disponível**, projetado especificamente para ser colocado **na frente** das tabelas do DynamoDB.

| DAX | ElastiCache |
| :---- | :---- |
| **Propósito** | Cache **dedicado** para DynamoDB, reduzindo latência de leituras para **microssegundos**. |
| **Integração** | **Compatível com as APIs DynamoDB existentes**. Requer pouca ou nenhuma alteração na lógica da aplicação (apenas troca o cliente DDB pelo cliente DAX). |
| **Hot Keys** | Resolve o problema de **hot keys** (itens muito populares), descarregando as leituras e evitando estrangulamento de RCU na tabela base. |
| **TTL Padrão** | Time-to-Live (TTL) padrão de **5 minutos** para dados em cache. |
| **Consistência** | Oferece consistência **final** para leituras em cache. |

---

### **10\. 🌊 DynamoDB Streams**

Streams do DynamoDB são um **log ordenado por tempo** de modificações de itens no nível da tabela (criações, atualizações e exclusões).

* **Retenção de Dados**: Os registros são retidos por **até 24 horas**. Para uma retenção mais longa, os dados devem ser processados e persistidos em outro serviço (ex: Kinesis Data Streams, S3).  
* **Usos Comuns**:  
  * **Reação em Tempo Real**: Acionar uma função AWS **Lambda** para reagir a alterações (ex: enviar um e-mail de boas-vindas após a criação de um usuário).  
  * **Análise/Busca**: Enviar dados para **Amazon OpenSearch Service** (para indexação de busca) ou **Amazon Redshift** (para análise).  
  * **Tabelas Derivadas**: Criar e manter uma cópia de dados em uma tabela diferente com um propósito específico.  
  * **Tabelas Globais**: Essencial para implementar a replicação entre regiões em **Global Tables**.  
* **Estrutura do Registro**: Você pode configurar quais informações aparecem no stream:  
  * **KEYS\_ONLY**: Apenas a chave primária.  
  * **NEW\_IMAGE**: A imagem completa do item após a modificação.  
  * **OLD\_IMAGE**: A imagem completa do item antes da modificação.  
  * **NEW\_AND\_OLD\_IMAGES**: Ambas as imagens, permitindo ver a mudança exata.  
* **Processamento com Lambda**: Uma **Event Source Mapping** interna da AWS extrai os registros em **batches** do Stream e invoca a função Lambda **sincronamente** com esses batches.

---

### **11\. ⏳ Time To Live (TTL)**

O recurso Time To Live (TTL) no DynamoDB permite a **exclusão automática** de itens expirados com base em um timestamp definido, sem consumir Unidades de Capacidade de Escrita (WCU).

* **Mecanismo**: Você designa um atributo (coluna) na sua tabela como o **atributo TTL**. O valor deste atributo deve ser um **número** que representa o **Unix Epoch Timestamp** (segundos desde 1º de janeiro de 1970).  
* **Processo de Exclusão**:  
  * O DynamoDB executa processos de varredura em segundo plano.  
  * Se o timestamp atual for **maior** que o valor do atributo TTL do item, o item é marcado para exclusão.  
  * A exclusão em si ocorre em um **período de tempo (geralmente até 48 horas)** após a expiração.  
* **Índices e Streams**: Quando um item é excluído pelo TTL, ele também é removido de todos os Índices Secundários Locais (LSI) e Globais (GSI). Além disso, a operação de exclusão é registrada no **DynamoDB Stream**.  
* **Itens Expirados em Consultas**: Itens expirados que ainda não foram excluídos pelo processo de fundo do DynamoDB **continuarão aparecendo** em operações de Read, Query e Scan. O filtro para esses itens precisa ser feito no lado do cliente.  
* **Casos de Uso**: É ideal para dados que só são relevantes por um período limitado, como **dados de sessão** (armazenamento de estado de sessão), logs ou informações que devem cumprir obrigações regulatórias de retenção mínima.

---

### **12\. 💻 Opções da CLI e Paginação**

Ao usar a CLI ou APIs para operações como Scan ou Query, existem opções cruciais para gerenciar a performance e a recuperação de dados.

* **\--projection-expression**: Especifica quais **atributos** (colunas) devem ser recuperados na consulta. Isso reduz a quantidade de dados transferidos e processados.  
* **\--filter-expression**: Filtra os itens **após** a leitura inicial da tabela, no lado do servidor, mas antes de serem retornados. É importante lembrar que o consumo de RCU é feito pela leitura de **todos os itens** antes da aplicação do filtro, tornando-o ineficiente para grandes conjuntos de dados se não for combinado com uma Query eficiente (no Partition Key).  
* **Paginação para Otimização da CLI**:  
  * **\--page-size**: Define o tamanho das páginas (lotes) de dados retornados a cada chamada de API interna. Mesmo que você queira o *dataset* completo, um tamanho de página menor pode **prevenir timeouts** em grandes operações de Scan ou Query ao dividir a carga em chamadas menores. O comando CLI principal ainda retorna o *dataset* inteiro.  
  * **\--max-items**: Limita o **número total de itens** exibidos no resultado final do comando CLI.  
  * **\--starting-token / NextToken**: Usado em conjunto com \--max-items para implementar a paginação real. O NextToken é retornado na saída quando há mais resultados. Ele deve ser passado como o valor de \--starting-token no próximo comando para recuperar o próximo lote de resultados.

---

### **13\. 🛡️ Transações DynamoDB**

Transações fornecem operações de **tudo-ou-nada** (all-or-nothing) para múltiplos itens, potencialmente em múltiplas tabelas, garantindo as propriedades **ACID** (Atomicidade, Consistência, Isolamento e Durabilidade).

* **Tipos de Consistência/Escrita**:  
  * **Leitura**: Além da Leitura Eventualmente Consistente e Fortemente Consistente, existe a **Consistência Transacional**, que garante uma visão consistente dos dados em todas as tabelas envolvidas na transação.  
  * **Escrita**: Existe a Escrita Padrão e a **Escrita Transacional**, onde todas as operações de escrita na transação devem ser bem-sucedidas, ou nenhuma delas será.  
* **Operações de API**:  
  * TransactGetItems: Executa uma ou mais operações GetItem transacionais.  
  * TransactWriteItems: Executa uma ou mais operações PutItem, UpdateItem e/ou DeleteItem transacionais.  
* **Custo**: As transações consomem **o dobro** das Unidades de Capacidade de Leitura e Escrita (**2x RCU/WCU**) que as operações padrão, devido ao processo de preparação e commit de duas fases.  
* **Cálculo de Capacidade (Exame)**: Ao calcular RCU/WCU para transações, você aplica o fator de multiplicação de **2x** **após** determinar a capacidade base (considerando arredondamento para cima e tamanho do item).  
  * *Exemplo WCU*: 3 escritas transacionais/s de 5KB $\\rightarrow$ $(3 \\times \\lceil 5/1 \\rceil) \\times 2 \= 30 \\text{ WCU}$.  
  * *Exemplo RCU*: 5 leituras transacionais/s de 5KB (arredonda para 8KB) $\\rightarrow$ $(5 \\times \\lceil 8/4 \\rceil) \\times 2 \= 20 \\text{ RCU}$.  
* **Casos de Uso**: Cenários que exigem alta integridade de dados, como **transações financeiras**, gerenciamento de inventário e jogos multiplayer.

### **14\. 🗃️ Armazenamento de Estado de Sessão (Comparação)**

O DynamoDB é uma ótima escolha para o armazenamento de estado de sessão devido à sua natureza **serverless** e escalabilidade automática.

| Serviço | Vantagens Chave | Desvantagens / Notas |
| :---- | :---- | :---- |
| **DynamoDB** | **Serverless**, escala automática (On-Demand), oferece durabilidade e TTL. | Não é "in-memory", tem latência um pouco maior que o ElastiCache. |
| **ElastiCache** (Redis/Memcached) | **"In-Memory"** (mais rápido), **latência muito baixa** (microssegundos). | Requer provisionamento e gerenciamento de nós de cache. |
| **EFS** (Elastic File System) | Armazenamento de arquivos **compartilhado** entre múltiplas instâncias EC2. | É um sistema de arquivos, não um banco de dados de chave-valor. |
| **EBS / Instance Store** | Armazenamento persistente (EBS) ou temporário (Instance Store). | **Somente para cache local**, não é compartilhado entre múltiplas instâncias. |
| **S3** | Armazenamento de objetos durável e barato. | **Latência alta** para armazenamento de sessão, inadequado para objetos pequenos e alta frequência de acesso/mudança. |

### **15\. 🔍 Write Sharding (Fragmentação de Escrita)**

Write Sharding é uma estratégia de design de esquema usada para **distribuir uniformemente** a carga de escrita (e leitura) quando a Chave de Partição original é limitada e gera "partições quentes" (*hot partitions*).

* **Problema (Partição Quente)**: Em uma aplicação de votação, se a candidate\_ID for usada como Partition Key, todos os votos se concentrarão em apenas duas partições (candidate\_A e candidate\_B), excedendo o throughput provisionado por partição.  
* **Solução**: Modificar a Partition Key adicionando um **prefixo ou sufixo aleatório** (Write Sharding).  
  * **Mecanismo**: A chave de partição se torna, por exemplo, candidate\_ID \+ \_N, onde N é um número aleatório ou calculado via *hashing*.  
  * **Resultado**: A Partition Key resultante terá uma **cardinalidade muito maior**, forçando o DynamoDB a distribuir os dados por um número maior de partições físicas, resolvendo o problema de concorrência e *hot partitions*.

---

### **16\. 📝 Tipos de Operações de Escrita**

É fundamental entender como o DynamoDB lida com a concorrência em operações de escrita.

* **Escritas Concorrentes (Concurrent Writes)**:  
  * **Comportamento Padrão**: Múltiplos usuários tentam atualizar o mesmo item simultaneamente. Todas as operações podem ser reportadas como sucesso, mas a **última escrita vence** (*Last Writer Wins*), e as escritas anteriores são sobrescritas, resultando em dados perdidos.  
* **Escritas Condicionais (Conditional Writes / Optimistic Locking)**:  
  * Resolve o problema de concorrência. Usa um atributo de **versão** ou outra condição para garantir que a escrita só ocorra se o item não tiver mudado desde que o cliente o leu. Se a condição falhar, a escrita é rejeitada.  
* **Escritas Atômicas (Atomic Writes)**:  
  * Garante que uma operação numérica (como incremento ou decremento) seja concluída sem que nenhuma outra escrita interfira no processo. Aumenta ou diminui um valor de atributo com segurança, garantindo que o resultado final seja a soma de todas as operações. Ex: Se um usuário aumenta em 1 e outro em 2, o valor final aumenta em 3, e nenhuma operação é perdida.

---

### **17\. ☁️ Sinergias DynamoDB e Amazon S3**

DynamoDB e S3 podem ser combinados para lidar com dados que o DynamoDB não suporta eficientemente (objetos grandes) ou para fornecer capacidades de consulta.

| Cenário | Estratégia |
| :---- | :---- |
| **Armazenamento de Objetos Grandes** | Armazene o objeto grande (imagem, vídeo, etc.) no **Amazon S3**. Armazene o **metadado** do objeto (ex: object\_key, image\_url) no DynamoDB. O item DynamoDB age como um **ponteiro**, permitindo que o DynamoDB mantenha o desempenho enquanto o S3 lida com grandes armazenamentos. |
| **Indexação de Metadados do S3** | Use o **DynamoDB** para indexar metadados de objetos armazenados no S3 (tamanho, data de criação, autor, etc.). O **S3 Event Notifications** pode acionar uma **função Lambda** para popular o DynamoDB quando novos objetos são carregados. Isso permite consultas complexas e eficientes no DynamoDB (ex: "encontrar todos os objetos de um cliente em um intervalo de datas") que seriam caras ou impossíveis de realizar diretamente no S3. |

---

### **18\. 🧹 Limpeza e Cópia de Tabelas**

* **Limpeza Rápida de Tabela**: A maneira mais rápida, eficiente e barata de limpar todos os dados de uma tabela é **deletar a tabela (DeleteTable) e recriá-la** com as mesmas configurações.  
  * **Evitar**: Fazer um Scan em toda a tabela e depois deletar os itens um por um é lento e consome muitas RCUs/WCUs.  
* **Cópia/Backup de Tabela**:  
  * **AWS Backup**: Serviço para backup e restauração em contas ou regiões diferentes.  
  * **AWS Glue**: Serviço ETL (Extract, Transform, Load) que pode ler a tabela de origem e escrever o conteúdo em qualquer destino (incluindo uma nova tabela DynamoDB ou S3).  
  * **Código Próprio**: Usar Scan (ou Query) seguido de PutItem ou BatchWriteItem (mais difícil e menos eficiente que os serviços AWS).

---

### **19\. 🔒 Segurança e Acessos Refinados (Fine-Grained Access Control)**

DynamoDB oferece um conjunto robusto de recursos de segurança.

* **Segurança Básica**:  
  * **Encryption at Rest**: Usando **AWS KMS**.  
  * **Encryption in Transit**: Usando **SSL/TLS**.  
  * **Conectividade Privada**: Através de **VPC Endpoints** (mantém o tráfego dentro da VPC e da rede AWS, evitando a internet pública).  
  * **Autorização Centralizada**: Totalmente controlada por **IAM**.  
* **Backup e Recuperação**:  
  * **Point-in-Time Recovery (PITR)**: Permite a restauração a **qualquer segundo** nos últimos 35 dias (sem impacto no desempenho).  
  * **Backup e Restauração sob Demanda**.  
* **Global Tables**: Fornecem tabelas **multi-região, multi-ativa** e totalmente replicadas. Exigem que o **DynamoDB Streams** esteja habilitado para gerenciar a replicação.  
* **Acesso Refinado (Fine-Grained Access Control)**: Permite que clientes web/mobile acessem diretamente o DynamoDB sem comprometer a segurança.  
  * **Mecanismo**:  
    1. **Login Federado**: Usuários se autenticam usando um Provedor de Identidade (ex: Amazon Cognito, Google, Facebook).  
    2. **Credenciais Temporárias**: As credenciais do provedor são trocadas por **credenciais AWS temporárias** (via STS ou Cognito Identity Pools).  
    3. **IAM Role com Condição**: O cliente assume um IAM Role que tem políticas com **condições** que restringem o acesso apenas aos dados que o usuário possui.  
  * **Restrições de Acesso Refinado**:  
    * **Nível de Linha (Item)**: Restringido por **LeadingKeys**. Isso limita o acesso aos itens onde o valor da Chave Primária (geralmente o Partition Key) corresponde à identidade do usuário.  
    * **Nível de Atributo (Coluna)**: Restringido por **Attributes** na política IAM. Isso limita quais atributos um usuário pode visualizar ou modificar em um item.
