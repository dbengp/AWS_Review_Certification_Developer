# 🚀 **AWS Lambda Functions**

## 1. Conceito e Objetivo Principal

O AWS Lambda é um serviço de **computação *serverless*** (sem servidor) que permite executar código (funções) em resposta a eventos sem a necessidade de provisionar ou gerenciar a infraestrutura subjacente (como servidores EC2).

* **Funções *Serverless***: Não há servidores para gerenciar. Você apenas carrega seu código e o Lambda cuida de tudo para executá-lo.  
* **Execução sob Demanda**: As funções são executadas somente quando são invocadas (sob demanda). **Você paga apenas pelo tempo de computação consumido** (em incrementos de 1 milissegundo) e pelo número de solicitações (invocações). Não há cobrança quando o código não está em execução.  
* **Escalabilidade Automática**: O AWS dimensiona automaticamente a concorrência e o número de ocorrências das funções para lidar com o volume de requisições, tornando-o altamente escalável.  
* **Limitação por Tempo**: As execuções são limitadas por tempo (atualmente, até **15 minutos** por execução). É ideal para execuções de curta e média duração.  
* **Recursos**: É possível provisionar até **10 GB de RAM** por função. **Aumentar a RAM também melhora o desempenho da CPU e da rede** da função, o que é um ponto importante para a prova.  
* **Runtimes (Linguagens)**: Suporta várias linguagens como **Node.js**, **Python**, Java, C\#, Ruby, e outras através do Custom Runtime API (como Rust ou Golang).  
* **Contêineres**: O Lambda suporta o empacotamento de funções como imagens de contêiner (Docker), mas é importante lembrar que para a maioria dos cenários de contêineres e para fins de exame, serviços como **ECS ou Fargate** são geralmente a opção preferida para gerenciar contêineres Docker tradicionais.

---

### **💰 Modelo de Preços**

O preço é baseado em:

1. **Número de Requisições (Invocações)**.  
2. **Duração** (tempo de execução) em incrementos de 1 ms, medido em **Gigabyte-segundos** (GB-segundo).  
   * **GB-segundo** \= Duração da execução (em segundos) $\\times$ Memória alocada (em GB).  
* **Nível Gratuito (*Free Tier*)** é muito generoso, incluindo:  
  * **1 milhão de requisições gratuitas** por mês.  
  * **400.000 GB-segundos** de tempo de computação por mês.

---

### **🔗 Integrações com Outros Serviços (Gatilhos)**

O Lambda se integra nativamente com muitos serviços AWS, permitindo criar arquiteturas reativas e orientadas a eventos:

* **API Gateway**: Criação de APIs REST que invocam funções Lambda.  
* **Amazon S3**: Eventos como upload de arquivo disparam uma função Lambda (ex: criação de miniaturas *thumbnails*).  
* **DynamoDB/Kinesis**: Criação de *triggers* para processar dados em tempo real ou reagir a mudanças no banco de dados.  
* **CloudWatch Events / EventBridge**:  
  * **CRON Job *Serverless***: Gatilho baseado em agendamento (taxa ou expressão cron) para executar tarefas periódicas.  
  * Reagir a eventos na infraestrutura AWS (ex: mudanças de estado do CodePipeline).  
* **SNS/SQS**: Processamento de mensagens ou reação a notificações.  
* **ALB (*Application Load Balancer*)**: Exposição de funções Lambda a *endpoints* HTTP/HTTPS, registrando a função em um **Grupo de Destino (*Target Group*)** do ALB.

---

### **🔄 Tipos de Invocação**

Existem duas formas principais de invocar funções Lambda:

#### **1\. Invocação Síncrona (RequestResponse)**

* **O cliente espera pelo resultado**. O Lambda executa a função e envia a resposta de volta ao invocador.  
* **Tratamento de Erros**: O cliente é responsável por gerenciar *timeouts* e **novas tentativas (*retries*)** em caso de falha.  
* **Serviços Síncronos**: CLI/SDK, API Gateway, ALB, Lambda@Edge, Cognito, Step Functions.  
  * **ALB/Lambda**: O ALB traduz a requisição HTTP em um documento **JSON** para o Lambda e converte a resposta JSON do Lambda de volta para HTTP. O recurso **Multi-Value Headers** do ALB pode ser habilitado para preservar múltiplos valores para a mesma chave de cabeçalho ou parâmetro de *query string* na *payload* do JSON.

#### **2\. Invocação Assíncrona (Event)**

* **O cliente não espera pelo resultado**. O evento é colocado em uma **Fila de Eventos Interna** do Lambda, e o Lambda retorna a confirmação de aceitação imediatamente.  
* **Tratamento de Erros**: O **serviço Lambda gerencia as novas tentativas**. Por padrão, ocorrem **3 tentativas** (a primeira, depois de 1 minuto, e depois de mais 2 minutos).  
* **Importância da Idempotência**: Devido às novas tentativas, a função Lambda deve ser **idempotente** (executar várias vezes o mesmo evento deve produzir o mesmo resultado).  
* **DLQ (*Dead-Letter Queue*)**: Se todas as tentativas falharem, o evento pode ser enviado para um **SQS** (Fila) ou **SNS** (Tópico) para processamento posterior e evitar a perda de dados.  
* **Serviços Assíncronos**: Amazon S3 Event Notifications, SNS, CloudWatch Events/EventBridge.

---

## **☁️ Integração com Eventos S3 (Invocação Assíncrona)**

A integração com o Amazon S3 é um padrão clássico de **Invocação Assíncrona** do Lambda.

* **Gatilho (*Trigger*)**: O S3 pode enviar notificações de eventos (ex: criação, remoção ou restauração de objetos) diretamente para o Lambda, SQS ou SNS.  
* **Caso de Uso Comum**: Geração de miniaturas (*thumbnails*) a partir de imagens recém-carregadas no S3.  
* **Configuração**: Você define a notificação de evento no *Bucket* S3, especificando:  
  * **Tipo de Evento**: Por exemplo, s3:ObjectCreated:\*.  
  * **Filtros**: Por **prefixo** (caminho) e **sufixo** (extensão do arquivo).  
  * **Destino**: A função Lambda a ser invocada.  
* **Permissões**: Para que o S3 possa invocar a função Lambda, o **S3 Service Principal** deve ter uma política de permissão baseada em recurso (*Resource-Based Policy*) anexada à função Lambda. Esta política permite que o *bucket* específico invoque a função.  
* **Garantia de Eventos**: Para garantir que você não perca notificações, especialmente em caso de múltiplas gravações simultâneas no mesmo objeto, é recomendável **habilitar o *versioning*** (controle de versão) no *bucket*.  
* **Processamento**: A função Lambda recebe um documento JSON com os detalhes do evento (nome do *bucket*, chave do objeto, tamanho, etc.) e pode usar essa informação para, por exemplo, fazer uma chamada GetObject no S3 e processar o arquivo.

---

## **➡️ Mapeamento de Fonte de Eventos (*Event Source Mapping*)**

O Event Source Mapping (ESM) é o mecanismo do Lambda para consumir registros de serviços de fluxo (*streams*) e filas (*queues*) que exigem **polling** (consultas contínuas). É o serviço Lambda que faz o *polling* e **invoca a função Lambda de forma síncrona** com um lote (*batch*) de registros.

| Serviço | Tipo de Fonte | Invocação | Observações Chave |
| :---- | :---- | :---- | :---- |
| **Kinesis Data Streams** | Streams | Síncrona | Processamento **ordenado** no nível do **shard**. |
| **DynamoDB Streams** | Streams | Síncrona | Processamento **ordenado** no nível do **shard**. |
| **SQS Standard** | Queues | Síncrona | Processamento **não ordenado**. Escala rapidamente (até 1000 lotes/segundo). |
| **SQS FIFO** | Queues | Síncrona | Processamento **ordenado** pelo MessageGroupID. Escala pelo número de grupos ativos. |

### **A\. Fontes de *Stream* (Kinesis e DynamoDB Streams)**

* **Polling e Processamento**: O ESM cria um iterador para cada *shard* do *stream* e lê os itens em lotes. Os itens são processados **em ordem** dentro de cada *shard*.  
* **Retenção de Dados**: Os registros **não são removidos** do *stream* após o processamento, permitindo que outros consumidores leiam os mesmos dados.  
* **Controle de Leitura**: É possível configurar o ponto de partida da leitura: a partir dos itens mais recentes (LATEST), do início (TRIM\_HORIZON) ou de um carimbo de data/hora específico.  
* **Tratamento de Erros**: Se a função falhar, **todo o lote (*batch*) é reprocessado**. Para garantir o processamento ordenado, o processamento daquele *shard* é **pausado**.  
  * Para mitigar bloqueios, você pode configurar o ESM para **descartar eventos antigos**, **limitar o número de *retries*** ou **dividir o lote em erro** (*split the batch on errors* \- útil se o *timeout* do Lambda for atingido).  
* **Paralelização**: É possível aumentar o paralelismo de processamento de um *shard* para até **10 lotes simultâneos** por *shard*. Mesmo com paralelização, a ordem é garantida no nível da **Chave de Partição (*Partition Key*)**.

### **B\. Fontes de Fila (SQS Standard e FIFO)**

* **Polling Eficiente**: O ESM consulta o SQS usando **Long Polling**.  
* **Configuração**: Você define o **tamanho do lote (*Batch Size*)** de mensagens a serem enviadas ao Lambda.  
* **Permissões**: A função Lambda (através de seu **Role de Execução IAM**) deve ter permissão (sqs:ReceiveMessage, sqs:DeleteMessage, etc.) para interagir com a fila SQS.  
* **Mensagens Processadas**: Quando o Lambda processa um lote com sucesso, ele **exclui as mensagens da fila**.  
* **DLQ (*Dead-Letter Queue*)**: Para filas SQS, o DLQ deve ser configurado na **fila SQS de origem**, não na função Lambda. Isso ocorre porque o ESM faz uma invocação **síncrona**, e o DLQ do Lambda é apenas para invocações assíncronas.  
* **Recomendação de *Timeout***: O *Timeout* de Visibilidade da Fila SQS deve ser configurado para, no mínimo, **seis vezes o *timeout* da função Lambda**.  
* **Escalabilidade SQS Standard**: O Lambda escala rapidamente, adicionando até **16 instâncias por minuto** para processar a fila o mais rápido possível (limite de 1000 lotes processados simultaneamente).  
* **SQS FIFO**: Garante o processamento **ordenado** pelo MessageGroupID. O escalonamento é limitado ao número de **grupos de mensagens ativos**.

---

## **🧩 Objetos Event e Context**

Toda função Lambda recebe dois objetos principais como entrada, independentemente do runtime (Python, Node.js, etc.):

### **A\. Objeto Event**

* É um documento em formato **JSON** que contém os **dados que a função deve processar**.  
* Ele é criado pelo serviço de invocação (Gatilho), contendo todas as informações necessárias para reagir ao evento.  
* **Exemplos**: Se o S3 invocar o Lambda, o Event conterá o nome do *bucket*, a chave do objeto e o tipo de evento S3. Se for o EventBridge, conterá detalhes da regra e do evento agendado.  
* **Formato**: É convertido para um objeto nativo do runtime (ex: um dicionário em Python).

### **B\. Objeto Context**

* Fornece **metadados sobre a invocação e o ambiente de execução** da função.  
* Ele não contém dados para processamento do evento em si, mas sim informações sobre o ambiente.  
* **Exemplos de Dados**:  
  * AWS Request ID: ID de solicitação exclusivo para esta invocação.  
  * Function Name: Nome da função Lambda.  
  * Memory Limit: Limite de memória configurado (em MB).  
  * Log Group / Log Stream Name: Nomes associados ao CloudWatch Logs.  
  * Function ARN: ARN da função.

---

## **🎯 Lambda Destinations (Destinos)**

Lambda Destinations é um recurso introduzido em 2019 que permite **rastrear e rotear o resultado** de uma invocação para um destino específico, seja em caso de sucesso ou falha no processamento.

### **A\. Invocação Assíncrona (S3, SNS, EventBridge)**

Para invocações assíncronas, você pode configurar destinos tanto para **sucesso** quanto para **falha**.

* **Destinos Suportados**: SQS, SNS, outra Função Lambda, ou EventBridge Bus.  
* **Sucesso**: O payload de **sucesso** é enviado ao destino após o processamento bem-sucedido.  
* **Falha**: O payload de **falha** é enviado ao destino **somente após todas as tentativas (*retries*) serem esgotadas** (lembre-se que invocações assíncronas têm 3 tentativas automáticas por padrão).  
* **Substituição do DLQ (Asynchronous)**: Embora você possa usar ambos, o **Destinations é o recurso recomendado** em vez do DLQ (*Dead-Letter Queue*) para invocações assíncronas, pois oferece mais tipos de destino (incluindo outra Lambda ou EventBridge, além de SQS/SNS) e também permite rotear eventos de sucesso.  
* **Payload de Destino**: O destino recebe um JSON completo que inclui: o **contexto da solicitação**, o **registro do evento original** que foi processado e, no caso de sucesso, o **payload da resposta** da função, ou em caso de falha, a **mensagem de erro** e o *stack trace*.

### **B\. Event Source Mapping (Streams e Queues)**

Para *Streams* (Kinesis/DynamoDB) e Filas (SQS), o Destination só é usado em caso de **falha de processamento grave** que resulta no **descarte de um lote de eventos**.

* **Destinos Suportados**: Apenas SQS ou SNS.  
* **Uso**: Quando um *batch* falha repetidamente, em vez de bloquear o *shard* ou o processamento, ele pode ser **descartado** e enviado ao destino de falha.  
* **SQS Específico**: Se a fonte for SQS, você pode optar por usar o Destination de Falha do Lambda ou configurar o DLQ diretamente na **fila SQS de origem**.

---

## **🔑 Roles de Execução e Políticas Baseadas em Recurso**

A segurança e as permissões no Lambda são gerenciadas por dois mecanismos principais:

### **A\. Role de Execução (Execution Role \- IAM Role)**

* É uma **IAM Role** anexada à função Lambda.  
* Define o que a **função Lambda pode fazer** (*saída* da função).  
* **Aplica-se a**:  
  * **Acesso a Outros Serviços**: Ex: Adicionar logs ao **CloudWatch Logs** (coberto pela AWSLambdaBasicExecutionRole padrão).  
  * **Leitura de Dados**: Quando a **Lambda está lendo ativamente** de um serviço (via Event Source Mapping \- ESM). Ex: Ler mensagens de uma fila **SQS** ou registros de um *stream* **Kinesis**.  
  * **Escrita em Destinos (*Destinations*)**: O Role precisa de permissão para enviar o resultado (sucesso/falha) aos destinos configurados (ex: sqs:SendMessage).  
  * **Acesso à VPC**: Se a função for configurada em uma VPC, o Role precisará da AWSLambdaVPCAccessExecutionRole para gerenciar as interfaces de rede.  
* **Melhor Prática**: Criar um Role de Execução específico para cada função.

### **B\. Políticas Baseadas em Recurso (*Resource-Based Policy*)**

* É uma política JSON anexada **DIRETAMENTE à função Lambda** (o recurso).  
* Define **quem pode invocar a função** (*entrada* da função).  
* **Aplica-se a**: **Comunicação entre serviços** (Service-to-Service).  
  * Ex: O **Amazon S3** precisa de uma *Resource-Based Policy* que o autorize a invocar o Lambda quando um arquivo é carregado.  
  * Ex: O **EventBridge** precisa dessa política para invocar o Lambda em um agendamento ou em resposta a um evento.  
* **Diferença SQS**: No caso do SQS/Kinesis (ESM), a **Política Baseada em Recurso não é necessária** porque o serviço Lambda é quem está *lendo* os dados, e não SQS/Kinesis quem está *invocando* a função (o Role de Execução cuida disso).

| Serviço de Invocação | Ação necessária | Tipo de Permissão | Quem faz a Ação? |
| :---- | :---- | :---- | :---- |
| **S3, SNS, EventBridge** | Invocar a função | Resource-Based Policy | O serviço S3/SNS/EventBridge invoca a função. |
| **SQS, Kinesis, DynamoDB** | Ler/Receber dados | Execution Role | O serviço Lambda (ESM) lê os dados do SQS/Kinesis. |

---

## **⚙️ Configuração e Implantação do Lambda**

### **1\. Variáveis de Ambiente (*Environment Variables*)**

* **Conceito**: Pares chave-valor em formato string que ajudam a ajustar o comportamento da função **sem a necessidade de alterar ou implantar o código**.  
* **Uso**: Ideais para configurar endpoints de banco de dados, chaves de API, ou para indicar o ambiente de execução (dev, prod).  
* **Segurança**: Variáveis de ambiente podem ser **criptografadas** usando **AWS KMS** (Key Management Service) para armazenar valores secretos. A chave pode ser gerenciada pela AWS ou ser uma CMK (Customer Master Key) sua.  
* **Acesso no Código**: Para acessar uma variável de ambiente, você usa o pacote ou módulo do sistema operacional (ex: os.getenv('CHAVE') em Python).

---

## **📊 Logging, Monitoramento e Rastreamento**

### **A\. CloudWatch Logs**

* **Integração Automática**: Todos os logs de execução da função Lambda são automaticamente armazenados no **CloudWatch Logs**.  
* **Permissão**: O Role de Execução do Lambda deve incluir a política de permissão para escrever logs, o que é fornecido pela AWSLambdaBasicExecutionRole.

### **B\. Métricas do CloudWatch**

* **Monitoramento**: Métricas automáticas são exibidas no painel do Lambda e do CloudWatch.  
* **Métricas Chave**:  
  * **Invocations**: Número de vezes que a função foi chamada.  
  * **Duration**: Tempo de execução da função (média, máxima, mínima).  
  * **Errors / Success Rate**: Contagem de erros e taxa de sucesso.  
  * **Throttles**: Contagem de vezes que a função foi limitada por atingir o limite de concorrência.  
  * **Iterator Age**: (Para Streams Kinesis/DynamoDB) Mede o quão atrasada está a leitura dos dados do *stream* (lagging).  
  * **Concurrent Executions**: Nível de paralelismo.

### **C\. Rastreamento com X-Ray**

* **Ativação**: O rastreamento é ativado na configuração do Lambda (Active Tracing). Isso executa o **X-Ray Daemon** no ambiente de execução.  
* **Instrumentação do Código**: Você deve usar o **X-Ray SDK** no seu código para instrumentar chamadas a outros serviços e registrar segmentos de *trace*.  
* **Permissão**: O Role de Execução precisa da permissão para escrever dados no X-Ray, garantida pela política gerenciada AWSXRayDaemonWriteAccess.  
* **Benefício**: Permite visualizar o **Service Map** e rastrear o fluxo completo de uma solicitação por vários serviços AWS, ajudando a identificar gargalos de latência.  
* **Variável de Ambiente Chave**: A variável AWS\_XRAY\_DAEMON\_ADDRESS informa o endereço (IP e porta) para comunicação com o *daemon* do X-Ray.

---

## **🌐 Personalização e Execução na Edge (CloudFront)**

As *Edge Functions* permitem executar código em **Locais de Borda (*Edge Locations*)** do CloudFront (próximo aos usuários) para reduzir a latência. Existem dois tipos:

| Característica | CloudFront Functions | Lambda@Edge |
| :---- | :---- | :---- |
| **Linguagem** | JavaScript (leve) | Node.js e Python |
| **Escala** | Milhões de requisições/seg (Alta Escala) | Milhares de requisições/seg |
| **Tempo de Execução Máx.** | **\< 1 Milissegundo** (Funções muito rápidas) | Até 5-10 Segundos |
| **Eventos de Gatilho** | Apenas **Viewer Request/Response** | **Viewer Request/Response** E **Origin Request/Response** |
| **Acesso a Recursos** | Limitado (sem rede/acesso a disco) | Permite acesso a rede, SDK da AWS, CPU/Memória ajustável. |
| **Casos de Uso** | Normalização de Cache Key, Manipulação de Header/URL, Autorização JWT simples. | Autenticação completa, A/B Testing, Lógica de roteamento complexa, Transformação de imagem em tempo real, Acesso a outros serviços AWS (via SDK). |

* **Viewer Request/Response**: Antes/depois do CloudFront interagir com o visualizador.  
* **Origin Request/Response**: Antes/depois do CloudFront interagir com o *Origin* (servidor de origem).

---

## **💻 Networking e VPC**

Por padrão, a função Lambda é executada em uma VPC de propriedade da AWS e possui acesso à Internet e a serviços públicos da AWS (como DynamoDB, S3) via **Internet Pública**.

### **A\. Acesso a Recursos em sua VPC**

* **Problema**: Por padrão, o Lambda **não pode acessar recursos privados** dentro da sua VPC (Ex: RDS, EC2 em Subnet Privada).  
* **Solução**: Você deve configurar o Lambda para ser executado **DENTRO da sua VPC** (especificando a VPC ID, Subnets e Security Group).  
* **Mecanismo**: O Lambda cria uma **ENI (*Elastic Network Interface*)** em cada *Subnet* selecionada. O Role de Execução deve ter a permissão AWSLambdaVPCAccessExecutionRole para criar essas ENIs.  
* **Regras de SG**: O Security Group do Lambda deve ser autorizado a se comunicar com o Security Group do seu recurso (Ex: O SG do Lambda deve ser a *Source* permitida no SG do RDS).

### **B\. Acesso à Internet (para Lambdas em sua VPC)**

* **Importante para a Prova**: Uma função Lambda implantada em uma **Subnet Pública DENTRO da sua VPC NÃO tem acesso à Internet** (o Lambda não recebe um IP Público automaticamente, diferentemente de uma EC2).  
* **Solução para Acesso à Internet**: Se o Lambda precisar acessar APIs externas ou serviços públicos da AWS:  
  1. O Lambda deve ser implantado em uma **Subnet Privada**.  
  2. O tráfego de saída (Outbound) da *Subnet Privada* deve ser roteado para um **NAT Gateway** ou **NAT Instance** localizado em uma *Subnet Pública*.  
* **Acesso Privado a Serviços AWS (VPC Endpoints)**: Para acessar serviços públicos da AWS (como DynamoDB ou S3) de forma privada, **sem passar pelo NAT Gateway/Internet**, você pode usar **VPC Endpoints**.

**Exceção**: O **CloudWatch Logs** sempre funciona, mesmo que o Lambda esteja em uma *Subnet Privada* e não tenha acesso configurado à Internet/NAT.

---

## **🚀 Configuração e Desempenho do Lambda**

### **1\. Memória (RAM) e vCPUs**

* **Configuração**: Você define a quantidade de **RAM** (mínimo de 128 MB, máximo de 10 GB) em incrementos de 1 MB.  
* **Performance Implícita**: Aumentar a RAM **implicitamente aumenta a quantidade de poder de CPU (vCPUs)** alocado à sua função. Você não pode definir vCPUs diretamente.  
* **Ponto de Referência**: A função atinge o equivalente a um **vCPU completo em 1792 MB de RAM**. Acima disso, ela recebe múltiplos vCPUs, exigindo o uso de *multi-threading* no código para aproveitar totalmente o poder de processamento.  
* **Otimização de Custo/Performance**: Para aplicações *CPU-bound* (com muito cálculo), aumentar a RAM pode reduzir o tempo de execução e, consequentemente, o custo total de cobrança (*duração x preço*).

### **2\. Timeout**

* **Padrão**: O *timeout* padrão é de 3 segundos.  
* **Máximo**: Você pode configurar o *timeout* para até **900 segundos (15 minutos)**.  
* **Caso de Uso**: Funções que duram mais de 15 minutos são inadequadas para Lambda e devem ser consideradas para AWS Fargate, ECS ou EC2.

### **3\. Contexto de Execução (*Execution Context*)**

* **Reuso**: O ambiente de execução (*Execution Context*) é mantido por um tempo após a execução da função em antecipação a novas invocações.  
* **Melhor Prática de Código**:  
  * **Inicialize conexões caras** (ex: conexões de banco de dados, clientes HTTP/SDK) **FORA** do *handler* da função.  
  * Ao fazer isso, essas conexões podem ser **reutilizadas** em invocações subsequentes no mesmo *Execution Context* (o que é chamado de *warm start*), melhorando drasticamente a performance.

### **4\. Armazenamento Temporário (/tmp)**

* **Uso**: Diretório temporário que permite à função escrever arquivos.  
* **Capacidade**: Oferece até **10 GB** de espaço em disco para a função.  
* **Persistência**: O conteúdo em /tmp **pode ser mantido** entre invocações se o *Execution Context* for reutilizado, mas é **Efêmero** (não garantido).  
* **Observação**: Para **persistência permanente** e compartilhamento garantido entre instâncias, use **Amazon S3** ou **EFS**.

---

## **📦 Gerenciamento de Código e Dependências**

### **1\. Pacote de Implantação e SDK**

* **Estrutura**: O pacote de implantação (*Deployment Package*) deve ser um **arquivo ZIP** contendo seu código e todas as suas dependências externas.  
* **Tamanho**:  
  * Se for **menor que 50 MB**, pode ser enviado diretamente pelo console.  
  * Se for **maior que 50 MB** (até o limite de 250 MB descompactado), deve ser enviado para o **Amazon S3** e referenciado pelo Lambda.  
* **SDK da AWS**: O **AWS SDK** para o runtime específico já está **incluído** por padrão no ambiente Lambda. Não é necessário empacotá-lo com seu código.  
* **Bibliotecas Nativas**: Bibliotecas que exigem compilação nativa (*native libraries*) devem ser compiladas em um ambiente **Amazon Linux** antes de serem empacotadas.

### **2\. Lambda Layers (Camadas)**

* **Conceito**: Uma Layer é um arquivo ZIP que contém bibliotecas, dependências personalizadas ou até mesmo *runtimes* customizados.  
* **Reutilização**: Permite **externalizar dependências pesadas** para que possam ser reutilizadas por múltiplas funções Lambda, evitando o reempacotamento das mesmas dependências toda vez que o código principal for alterado.  
* **Benefício**: Reduz o tamanho do pacote de código da função (agilizando a implantação) e promove a reutilização de código entre funções.  
* **Limite**: Você pode anexar até **cinco Layers** por função (contribuindo para o limite total de 250 MB descompactado).

---

## **💾 Opções de Armazenamento para Lambda**

| Opção de Armazenamento | Capacidade Máxima | Persistência | Uso | Acesso | Velocidade |
| :---- | :---- | :---- | :---- | :---- | :---- |
| **/tmp (Ephemeral Storage)** | Até 10 GB | Efêmero (não garantido entre execuções) | Arquivos temporários, processamento em disco. | Local (apenas a função) | Mais Rápido |
| **Lambda Layers** | 250 MB (total com código) | Durável (Imutável/Somente leitura) | Bibliotecas, dependências reutilizáveis, runtimes customizados. | Compartilhado (entre invocações e funções) | Mais Rápido |
| **Amazon S3** | Ilimitado | Durável | Armazenamento de objetos, permanente. | API S3 (via rede) | Rápido (via rede dedicada) |
| **Amazon EFS (*Elastic File System*)** | Elástico (petabytes) | Durável | Compartilhamento de arquivos, acesso de sistema de arquivos (*mount*). | Sistema de Arquivos (via rede VPC) | Muito Rápido (rede VPC) |

* **EFS Mount**: O Lambda pode montar um sistema de arquivos EFS se estiver rodando em uma VPC. Ele usa **EFS Access Points** para gerenciar as permissões e a conexão. É ideal para compartilhar grandes volumes de dados ou para aplicações que exigem operações de sistema de arquivos.

---

## **🚦 Concorrência e *Throttling***

### **1\. Concorrência Reservada (*Reserved Concurrency*)**

* **Objetivo**: Limitar ou garantir um número mínimo de execuções simultâneas para uma função.  
* **Configuração**: Definida no nível da **função individual**.  
* **Impacto no Pool**: A concorrência reservada para uma função **reduz o limite disponível** para todas as outras funções na sua conta, garantindo que a função reservada sempre tenha a capacidade necessária.  
* **Throttling**: Se o limite de concorrência for atingido (seja o limite da conta ou a concorrência reservada), invocações adicionais resultam em *Throttling* (**Erro 429**).  
  * **Síncrono**: Retorna o erro 429 imediatamente.  
  * **Assíncrono**: O Lambda reenvia o evento para a fila interna e tenta novamente por até 6 horas, com *backoff* exponencial.

### **2\. Concorrência Provisionada (*Provisioned Concurrency*)**

* **Objetivo**: Eliminar **Cold Starts** (partidas a frio).  
* **Funcionamento**: Aloca e inicializa **instâncias de função em execução** antes que as invocações cheguem.  
* **Benefício**: Garante que as invocações tenham **baixa latência** e latência previsível, pois o código já está carregado e pronto para executar (*warm*).  
* **Gerenciamento**: Pode ser escalonada automaticamente usando o **Application Auto Scaling** com base em cronogramas ou métricas de utilização.

### **3\. Cold Starts**

* **Definição**: Ocorre na **primeira invocação** de uma nova instância de função, exigindo tempo para carregar o código, o *runtime* e rodar o código de inicialização fora do *handler*.  
* **Impacto**: Resulta em **alta latência** para o primeiro usuário a acessar a nova instância.  
* **Otimização**: A concorrência provisionada é a principal solução para mitigar *cold starts*, especialmente para aplicações sensíveis à latência. As melhorias na Lambda VPC reduziram drasticamente a latência de *cold start* em funções dentro de uma VPC.

---

## **🛠️ Implantação e Empacotamento**

### **1\. Implantação com CloudFormation**

CloudFormation pode ser usado para automatizar a criação e gestão de funções Lambda de duas maneiras:

* **Inline Code**: Para funções **muito simples** sem dependências, o código pode ser inserido diretamente no template CloudFormation usando a propriedade Code.ZipFile.  
* **Referência a S3**: A maneira recomendada para funções com dependências. O pacote .zip da função é armazenado no **Amazon S3** e referenciado no template do CloudFormation usando:  
  * S3Bucket: Nome do *bucket*.  
  * S3Key: Caminho completo do arquivo .zip.  
  * S3ObjectVersion: **Recomendado** usar o versionamento do S3. Se o código for sobrescrito no S3 sem atualizar esta versão no template, o CloudFormation **não atualizará** a função Lambda.  
* **Implantação Cross-Account**: Para que o CloudFormation (em Conta B) possa buscar o código Lambda (em S3 na Conta A), é necessário configurar:  
  * Uma **Bucket Policy** no S3 (Conta A) permitindo acesso.  
  * Uma **Execution Role** no CloudFormation (Conta B) permitindo s3:GetObject no *bucket* da Conta A.

---

### **2\. Imagens de Contêineres (Containers)**

Um recurso mais recente que permite empacotar o Lambda como uma **imagem de contêiner de até 10 GB** (o que é muito maior que o limite de 250 MB do zip).

* **Base Image**: A imagem Docker deve ser construída sobre uma **Base Image fornecida pela AWS** (ou uma customizada) que implemente a **Lambda Runtime API**.  
* **Workflow Unificado**: Permite usar um fluxo de CI/CD unificado (construir, enviar para **ECR**, implantar no Lambda/ECS) e empacotar dependências grandes e complexas (ex: Data Science).  
* **Otimização**:  
  * Usar *Base Images* da AWS otimiza o cache do serviço Lambda.  
  * Usar *multi-stage builds* (construções em múltiplos estágios) é a melhor prática para garantir que a imagem final seja a menor possível, descartando artefatos de construção intermediários.

---

## **🔁 Versões e Aliases**

Estes recursos permitem gerenciar o ciclo de vida da função de forma estruturada.

* **Versões (Versions)**:  
  * São **imutáveis** (código e configuração fixos).  
  * Criadas ao **Publicar** uma função (ex: V1, V2...).  
  * Cada versão tem um ARN exclusivo.  
  * A versão **$LATEST** é sempre a versão **mutável** (em desenvolvimento).  
* **Aliases (Apelidos)**:  
  * São **mutáveis** e atuam como **ponteiros** para uma ou mais versões.  
  * Permitem usar um endpoint estável (ARN do Alias) que pode ser reconfigurado no *backend* (ex: PROD Alias, DEV Alias).  
  * **Não podem referenciar outros Aliases** (somente versões).  
* **Implantação Canary (*Canary Deployments*)**:  
  * Aliases permitem **roteamento de tráfego ponderado** entre duas versões (ex: 95% para V1 e 5% para V2).  
  * Isso permite testar uma nova versão em produção com um subconjunto de tráfego antes de fazer a mudança completa.

### **3\. Integração com CodeDeploy**

O CodeDeploy automatiza o processo de mudança de tráfego ponderado (Canary ou Linear) para os Aliases do Lambda:

* **Estratégias**:  
  * **Linear**: Aumenta o tráfego gradualmente ao longo do tempo (ex: Linear10PercentEvery3Minutes).  
  * **Canary**: Muda uma pequena porcentagem do tráfego por um período e, em seguida, move o restante (ex: Canary10Percent5Minutes).  
  * **AllAtOnce**: Muda 100% do tráfego imediatamente (mais rápido, mas mais arriscado).  
* **Rollback**: O CodeDeploy usa **Hooks de Tráfego (*Pre/Post Traffic Hooks*)** ou **CloudWatch Alarms** para verificar a saúde da nova versão. Se a saúde falhar, o tráfego é automaticamente revertido (rollback) para a versão anterior estável.

---

## **🔗 Lambda Function URLs**

Permite expor a função Lambda como um **Endpoint HTTP dedicado** e exclusivo, sem a necessidade de uma API Gateway ou Load Balancer.

* **Acesso**: Endpoint HTTPS acessível **somente via Internet Pública**.  
* **Segurança**: Gerenciada por **Políticas Baseadas em Recurso (*Resource-Based Policies*)**.  
  * AuthType: NONE: Acesso público não autenticado. A *Resource Policy* deve permitir lambda:InvokeFunctionUrl para Principal: \*.  
  * AuthType: AWS\_IAM: Exige autenticação e autorização via IAM (tanto a *Identity Policy* quanto a *Resource Policy* são avaliadas).  
* **Cross-Domain**: Suporta configuração **CORS** para chamadas de diferentes domínios.

---

## **🧠 Integração com CodeGuru**

* **CodeGuru Profiler**: Fornece insights sobre o **desempenho em runtime** das funções Lambda (suportado para Java e Python).  
* **Mecanismo**: Ao ser ativada no console, adiciona automaticamente um **CodeGuru Profiler Layer** e a política IAM AmazonCodeGuruProfilerAgentAccess ao Role de Execução.

---

## **🛑 Limites Chave do Lambda (por Região)**

O conhecimento destes limites é crucial para a prova:

| Categoria | Limite | Notas |
| :---- | :---- | :---- |
| **Execução** | Memória (RAM) | 128 MB a **10 GB** (em incrementos de 1 MB). |
|  | Tempo de Execução Máx. | **900 segundos (15 minutos)**. |
|  | Concorrência | **1000 execuções simultâneas** (pode ser aumentada via solicitação). |
|  | Espaço Temporário (/tmp) | **10 GB** de armazenamento temporário em disco. |
| **Implantação** | Tamanho Máximo do Zip (Comprimido) | **50 MB** (upload direto). |
|  | Tamanho Máximo Descomprimido (Total) | **250 MB** (incluindo layers). |
|  | Variáveis de Ambiente | **4 KB** (tamanho total). |

### **💡 Melhores Práticas Finais**

* **Otimização do Cold Start**: Realize o **trabalho pesado fora do *handler*** da função (conexões DB, inicialização de SDKs).  
* **Variáveis de Ambiente**: Use para valores de configuração (*strings* de conexão, nomes de recursos). Criptografe dados sensíveis com KMS.  
* **Tamanho do Pacote**: Minimize o tamanho do pacote de implantação. Use **Lambda Layers** para dependências reutilizáveis.  
* **Evite Recursão**: Uma função Lambda **nunca** deve chamar a si mesma, pois isso pode levar a um loop infinito e custos elevados.

