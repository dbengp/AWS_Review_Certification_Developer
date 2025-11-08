# **Cloud Formation**

## 1. Conceito e Objetivo Principal

AWS CloudFormation é um dos serviços mais importantes da AWS que implementa o conceito de **Infraestrutura como Código (IaC)**. Ele permite que você **descreva** (declare) sua infraestrutura e recursos da AWS (como EC2, S3, Load Balancers, etc.) usando código em um **template** (modelo), e o CloudFormation se encarrega de provisionar e configurar esses recursos na ordem correta.

CloudFormation opera removendo a necessidade de trabalho manual de configuração de recursos na console da AWS.

| Conceito | Descrição |
| :---- | :---- |
| **Template (Modelo)** | Um arquivo (geralmente **YAML** ou **JSON**) que declara de forma textual todos os recursos da AWS que se deseja criar, suas configurações e interconexões. |
| **Stack (Pilha)** | A coleção de recursos da AWS que são criados, atualizados e gerenciados como uma **única unidade** com base em um template. |
| **Infraestrutura como Código (IaC)** | O princípio fundamental. Todos os recursos são provisionados via código, o que melhora o **controle** e permite o uso de **controle de versão** (ex: Git) para gerenciar alterações na infraestrutura. |
| **Programação Declarativa** | Você não precisa definir a **ordem** de criação (orquestração) dos recursos; você apenas declara o que quer, e o CloudFormation se encarrega de descobrir as dependências e a sequência correta de criação/atualização. |

---

## **2\. Benefícios Chave do Uso do CloudFormation**

O uso do CloudFormation oferece ganhos em várias dimensões:

* **Controle e Revisão:** Nenhuma criação de recurso é manual. Todas as alterações de infraestrutura são revisadas através de mudanças no código.  
* **Produtividade:** Capacidade de **destruir e recriar** infraestruturas completas rapidamente (*on the fly*), maximizando o poder do *pay-as-you-go* da nuvem.  
* **Custo:**  
  * Todos os recursos em uma Stack são **automaticamente tagueados** com um identificador, facilitando o acompanhamento dos custos.  
  * Permite estratégias de economia, como automatizar a exclusão de ambientes de desenvolvimento em horários específicos e a recriação segura no dia seguinte.  
* **Reutilização:** Evita "reinventar a roda", permitindo o aproveitamento de templates existentes e da documentação para criação rápida.  
* **Separação de Preocupações:** Possibilidade de criar múltiplas Stacks para diferentes camadas ou aplicações (ex: uma Stack para a **rede/VPC** e outra Stack para os **aplicativos**).  
* **Visualização:** Geração automática de diagramas arquitetônicos a partir do template (ex: usando o Application Composer).

---

## **3\. Funcionamento e Ciclo de Vida do Stack**

### **Criação e Implantação**

1. O **Template** (arquivo YAML/JSON) deve ser carregado (upload) para o **Amazon S3**.  
2. O serviço CloudFormation faz uma **referência** a este arquivo S3.  
3. Com base nesse template, uma **Stack** é criada.

### **Atualização**

* **Não** é possível editar um template existente.  
* Para atualizar, você deve fazer o **upload de uma nova versão** do template (com as alterações) e usá-la para atualizar a Stack.  
* **Change Set (Conjunto de Mudanças):** Antes de aplicar a atualização, o CloudFormation gera um *preview* (visualização) das mudanças que serão feitas.  
  * O Change Set indica se um recurso será **adicionado**, **modificado** (mudança *in-place*) ou **substituído** (Replacement: true – o recurso antigo é terminado e um novo é criado para substituí-lo).  
* Se a atualização falhar, o CloudFormation tentará um **rollback** para o estado anterior da Stack.

### **Limpeza (Exclusão)**

* Ao **deletar uma Stack** a partir do CloudFormation, todos os artefatos e **todos os recursos** que foram criados por ela são **automaticamente excluídos**.  
* O CloudFormation é capaz de determinar a ordem correta para exclusão dos recursos para garantir uma limpeza completa.

---

## **4\. Resources (Recursos): O Core do Template**

A seção **Resources** (Recursos) é a única seção **obrigatória** em qualquer template do CloudFormation e constitui o núcleo da infraestrutura a ser provisionada.

### **Declaração e Tipos**

* **Finalidade:** Os Recursos representam os componentes reais da AWS (ex: instâncias EC2, Security Groups) que serão criados e configurados.  
* **Referências:** Os recursos podem fazer **referência** uns aos outros, e a AWS CloudFormation se encarrega de descobrir as dependências e a ordem correta para criação, atualização e exclusão.  
* **Identificação do Tipo:** Os identificadores de tipo de recurso seguem o formato: provedor-de-serviço::nome-do-serviço::nome-do-tipo-de-dados.  
  * *Exemplo:* AWS::EC2::Instance

### **Documentação e Propriedades**

* **Propriedades:** Cada tipo de recurso possui uma lista vasta de **Propriedades** (Properties) configuráveis. O que pode ser especificado na Console da AWS geralmente pode ser especificado via CloudFormation.  
* **Como Ler a Documentação:** A documentação oficial (disponível publicamente) é crucial. Ao consultar um tipo de recurso (ex: AWS::EC2::Instance), é possível ver a sintaxe (YAML ou JSON) e a lista de todas as propriedades disponíveis.  
  * A documentação detalha se uma propriedade é **obrigatória** (Required), qual é o seu **tipo** (ex: String, Array) e como a alteração dessa propriedade afeta a Stack durante uma atualização (ex: Requires No Interruption, ou Requires Replacement).

### **Limitações e Soluções (FAQ)**

* **Criação Dinâmica:** A criação dinâmica de um número variável de recursos (em *loop*) não é nativa do escopo básico do CloudFormation, exigindo o uso de **Macros** e **Transformações** avançadas.  
* **Suporte a Serviços:** Quase todos os serviços da AWS são suportados. Para lidar com serviços não suportados ou funcionalidades de terceiros, o mecanismo esperado (inclusive em contextos de certificação) é o uso de **CloudFormation Custom Resources** (Recursos Personalizados).

---

## **5\. Parameters (Parâmetros): Inputs Dinâmicos**

Parameters são utilizados para fornecer **inputs dinâmicos** aos templates do CloudFormation no momento da criação ou atualização de uma Stack.

### **Utilidade**

* **Reutilização:** Permite que o mesmo template seja reutilizado em diferentes contextos (ex: diferentes ambientes ou contas) sem alterar o seu código, apenas o valor do parâmetro.  
* **Configuração *Runtime*:** Ideal para valores que **não podem ser determinados antecipadamente** ou que são **propensos a mudar** no futuro. Se um valor for alterado, basta atualizar a Stack com o novo valor do parâmetro, sem a necessidade de carregar uma nova versão do template.

### **Configurações Importantes**

* Os parâmetros podem ser definidos com diferentes **tipos** (String, Number, CommaDelimitedList, List of Numbers) ou tipos específicos da AWS (ex: AWS::EC2::Instance::Type) para ajudar na validação.  
* **Validação e Controle:** É possível definir **restrições** e validações, como:  
  * **AllowedValues:** Limita a entrada a uma lista predefinida (ex: t2.micro, t2.small). Isso oferece escolha ao usuário, mas mantém o controle do administrador.  
  * **MaxLength/MinLength, MaxValue/MinValue, AllowedPattern (regex).**  
* **Segurança (NoEcho):** Se o valor for sensível (ex: senha de banco de dados), NoEcho: true impede que o valor seja exibido em logs ou na console, mantendo o segredo.

### **Uso dos Parâmetros (\!Ref)**

* O valor de um Parâmetro é acessado em outras seções do template (como Resources) usando a função de referência (Fn::Ref) ou sua forma abreviada no YAML: **\!Ref**.  
  YAML  
  GroupDescription: \!Ref SecurityGroupDescription

### **Parâmetros Pseudônomos (Pseudo Parameters)**

São variáveis **predefinidas** fornecidas pela AWS que podem ser referenciadas a qualquer momento, sem a necessidade de serem criadas no template. Exemplos importantes incluem:

* AWS::AccountId (ID da sua conta AWS).  
* AWS::Region (A região em que a Stack está sendo criada).  
* AWS::StackName (O nome da Stack atual).  
* Esses parâmetros permitem que o template se torne autoconsciente sobre seu contexto de implantação.

---

## **6\. Mappings (Mapeamentos): Variáveis Estáticas e Fixas**

Mappings são usados para definir **variáveis fixas e hardcoded** dentro do template. Eles são úteis para diferenciar configurações com base em ambientes (dev vs. prod) ou, mais comumente, por região.

### **Estrutura e Acesso**

* Mappings armazenam valores em uma estrutura aninhada, frequentemente utilizada como um mapa de regiões.  
* **Exemplo:** Um RegionMap pode associar diferentes IDs de AMI (que são específicos de cada região) à região de implantação e ao tipo de arquitetura.  
* **Função de Acesso (Fn::FindInMap):** Para obter um valor de um Mapping, utiliza-se a função Fn::FindInMap (ou \!FindInMap no YAML), que requer três chaves:  
  1. O nome do Mapa (ex: RegionMap).  
  2. A chave de nível superior (ex: \!Ref AWS::Region).  
  3. A chave de nível secundário (ex: HVM64).

### **Mappings vs. Parameters**

| Recurso | Quando Usar |
| :---- | :---- |
| **Mappings** | Quando **você sabe antecipadamente** todos os valores possíveis, e eles podem ser deduzidos de variáveis fixas (Região, Conta, Ambiente). Oferece maior controle de segurança. |
| **Parameters** | Quando o valor depende do que o **usuário deseja** em tempo de execução e não pode ser predefinido. Oferece mais liberdade ao usuário. |

---

## **7\. Outputs (Saídas): Ligando Stacks**

A seção **Outputs** (Saídas) é opcional e serve para declarar valores de recursos que foram criados, permitindo que eles sejam visualizados na console ou, mais crucialmente, **reutilizados por outras Stacks**.

### **Colaboração e Reutilização**

* **Visualização:** Os valores de saída (Outputs) facilitam a obtenção de informações importantes, como IDs de recursos (ex: ID da VPC ou ID de um Security Group) após a criação da Stack.  
* **Colaboração entre Stacks:** O principal caso de uso é a colaboração. Uma "Stack de Rede" pode exportar o ID da VPC e os IDs das Subnets.  
  * **Exportação:** Um Output pode ter um bloco Export, que define um Name exclusivo na região.  
  * **Importação:** Outra "Stack de Aplicação" pode consumir este valor exportado usando a função **Fn::ImportValue** (ou \!ImportValue).  
    YAML  
    SecurityGroups:  
      \- \!ImportValue SSHSecurityGroup

* **Gerenciamento de Dependência:** Uma Stack que exporta um valor **não pode ser excluída** enquanto houver outra Stack importando esse valor, garantindo a integridade e a ordem de exclusão dos recursos interligados.

---

## **8\. Conditions (Condições)**

A seção **Conditions** permite controlar a criação de recursos ou outputs com base em uma lógica condicional específica.

* **Propósito:** Definir uma lógica (booleana) que determina se um recurso deve ou não ser provisionado. Isso é essencial para diferenciar ambientes (ex: **Dev**, **Test**, **Prod**) ou regiões.  
  * *Exemplo:* Criar um recurso mais caro (como um volume EBS) apenas se o ambiente selecionado for Prod.  
* **Definição:** Uma Condição é definida usando **Funções Intrínsecas** de condição, como Fn::Equals, Fn::Not, Fn::And, Fn::Or, e Fn::If.  
  * As condições podem referenciar **Parâmetros** (para verificar o ambiente selecionado) ou **Mappings**.  
* **Aplicação:** A condição é aplicada diretamente ao Recurso ou Output usando a propriedade Condition: NomeDaCondição. Se a condição for avaliada como **verdadeira** (true), o recurso é criado; se for **falsa** (false), ele é ignorado.

---

## **9\. Funções Intrínsecas (Intrinsic Functions)**

As Funções Intrínsecas são funções integradas que o CloudFormation utiliza para atribuir valores dinâmicos, fazer referências e manipular dados dentro dos templates. As mais importantes (e já vistas) são: \!Ref, \!FindInMap, e \!ImportValue.

### **Funções Essenciais Adicionais**

| Função | Finalidade | Detalhes |
| :---- | :---- | :---- |
| **Fn::GetAtt** (\!GetAtt) | Obtém um **atributo** específico de um recurso que foi criado. | Usada para recuperar informações além do ID (que é o padrão do \!Ref), como AvailabilityZone, PrivateIp, ou PublicDNSName de uma instância EC2. |
| **Fn::Base64** (\!Base64) | Codifica um valor de string para o formato **Base64**. | O principal uso é codificar o *User Data* de uma instância EC2, que deve ser transmitido nesse formato. |
| **Funções de Condição** | Permitem avaliar condições booleanas. | Incluem Fn::And, Fn::Equals, Fn::If, Fn::Not e Fn::Or. |
| **Outras** | **Fn::Join**, **Fn::Sub**, **Fn::Split**, etc. | Funções de manipulação de string e lógica que podem ser consultadas na documentação oficial. |

### **Comparação \!Ref vs. \!GetAtt**

* **\!Ref Recurso:** Retorna o **ID físico** ou o valor de referência principal do recurso (ex: o ID da instância EC2).  
* **\!GetAtt Recurso.Atributo:** Retorna um atributo específico do recurso (ex: \!GetAtt EC2Instance.AvailabilityZone). Os atributos disponíveis estão sempre listados na documentação do tipo de recurso.

---

## **10\. Rollbacks (Reversões)**

Rollbacks referem-se ao mecanismo de segurança do CloudFormation que tenta reverter a Stack para um estado estável em caso de falha de criação ou atualização.

### **10.1 Falha de Criação (Stack Creation Failure)**

* **Comportamento Padrão:** O padrão é **fazer o rollback de tudo**, ou seja, todos os recursos que foram criados com sucesso antes da falha são **excluídos**. Isso garante uma limpeza completa (o estado volta a ser "sem Stack").  
* **Opção de *Troubleshooting*:** É possível **desabilitar o rollback** (Preserve successfully provisioned resources). Neste caso, os recursos criados com sucesso são mantidos para fins de **diagnóstico** e solução de problemas, mas a Stack permanece em um estado de falha (CREATE\_FAILED).

### **10.2 Falha de Atualização (Stack Update Failure)**

* **Comportamento Padrão:** Em caso de falha durante uma atualização, o CloudFormation tentará **automaticamente o rollback** para o último estado de trabalho **estável** conhecido. Qualquer recurso recém-criado ou modificado durante a atualização com falha é excluído ou revertido.

### **10.3 Falha de Rollback**

* Se o processo de rollback em si falhar (muitas vezes devido a recursos que foram **modificados manualmente** fora do CloudFormation), a Stack entra em um estado problemático (UPDATE\_ROLLBACK\_FAILED).  
* **Correção:** É necessário **corrigir o recurso manualmente** (fora do CloudFormation) para resolver a causa da falha. Em seguida, use a chamada de API/CLI **ContinueUpdateRollback** para instruir o CloudFormation a tentar a reversão novamente e alcançar um estado estável.

---

## **11\. Service Roles (Funções de Serviço) para Segurança**

CloudFormation pode usar uma **Função de Serviço (Service Role) do IAM** que você define, em vez de usar as permissões do usuário que está chamando a operação.

* **Princípio do Menor Privilégio:** Este mecanismo é fundamental para implementar o princípio do **menor privilégio**. O usuário pode não ter permissão direta para criar um bucket S3, mas pode criar um Stack que cria um S3, desde que ele tenha duas permissões:  
  1. Permissão para fazer ações no serviço CloudFormation.  
  2. Permissão **iam:PassRole** para "passar" a Função de Serviço dedicada ao CloudFormation.  
* **O Service Role:** A Função de Serviço é um **IAM Role** dedicado ao CloudFormation, e é este Role que detém as permissões reais (ex: S3:\*, EC2:\*) necessárias para criar, atualizar e excluir os recursos.  
* **Aplicação:** O Service Role é especificado na seção **Permissões (Permissions)** ao iniciar a criação ou atualização de uma Stack. Se não for especificado, o CloudFormation usará as permissões do usuário logado.

---

## **12\. Capabilities (Capacidades)**

As Capabilities (Capacidades) são parâmetros que você deve **explicitamente reconhecer** ao implantar ou atualizar um template. Elas servem como uma medida de segurança para garantir que você está ciente de que o template executará ações específicas e potencialmente sensíveis.

| Capacidade | Finalidade |
| :---- | :---- |
| **CAPABILITY\_IAM** | Deve ser fornecida quando o template cria ou atualiza recursos **IAM** (ex: usuários, grupos, políticas, roles) que **não** possuem um nome lógico definido no template. |
| **CAPABILITY\_NAMED\_IAM** | Deve ser fornecida quando o template cria ou atualiza recursos **IAM** que **possuem nomes personalizados** (definidos) no template. |
| **CAPABILITY\_AUTO\_EXPAND** | Necessária quando o template inclui **Macros** ou **Nested Stacks** (Stacks aninhadas) que realizam transformações dinâmicas no template antes da implantação. |

Se você tentar implantar um template que exige uma dessas capacidades sem reconhecê-la, receberá uma exceção (InsufficientCapabilitiesException), e a submissão falhará.

---

## **13\. DeletionPolicy (Política de Exclusão)**

A DeletionPolicy é uma configuração aplicada a recursos individuais dentro de um template para controlar o que acontece com eles quando são **removidos do template** ou quando a **Stack é excluída**. O comportamento padrão é Delete.

| Política | Ação na Exclusão da Stack/Recurso | Casos de Uso |
| :---- | :---- | :---- |
| **Delete** (Padrão) | O recurso é **excluído** juntamente com a Stack. | Aplicável à maioria dos recursos. |
| **Retain** | O recurso é **preservado** (mantido) na sua conta AWS após a exclusão da Stack. | Usado para proteger dados, como Security Groups ou DynamoDB Tables, que não devem ser apagados acidentalmente. |
| **Snapshot** | Cria um **snapshot final** do recurso antes de excluí-lo. O recurso em si é excluído, mas o snapshot permanece. | Suportado por bancos de dados e volumes (RDS, EBS, ElastiCache, etc.), sendo muito útil para fins de backup e segurança. |

⚠️ **Exceção S3:** Um bucket S3 com DeletionPolicy: Delete só será excluído se estiver **vazio**. Se o bucket não estiver vazio, a exclusão da Stack falhará, a menos que você use um **Recurso Personalizado** para esvaziá-lo primeiro (ver Seção 17).

---

## **14\. Stack Policies (Políticas de Stack)**

As Stack Policies são documentos JSON que definem quais ações de **atualização** (Update) são permitidas em recursos específicos de uma Stack.

* **Propósito:** Proteger recursos críticos (como um "Production Database") contra atualizações **não intencionais** que possam levar a *downtime* ou perda de dados.  
* **Mecanismo:** Elas são anexadas à Stack e definem regras de Allow (permitir) e Deny (negar) para ações de atualização.  
  * Por padrão, ao definir uma Stack Policy, todos os recursos são protegidos, exigindo um Allow explícito para os recursos que você deseja que possam ser atualizados.

---

## **15\. Termination Protection (Proteção contra Encerramento)**

Termination Protection é uma configuração simples aplicada diretamente à **Stack inteira** para prevenir sua exclusão acidental.

* **Função:** Quando ativada, impede que a Stack seja deletada.  
* **Uso:** É uma medida de segurança contra exclusões acidentais de ambientes de produção inteiros. Para deletar a Stack, o usuário deve **primeiro desativar** a proteção contra encerramento (o que exige permissões específicas).

---

## **16\. Custom Resources (Recursos Personalizados)**

Custom Resources são uma funcionalidade avançada que permite provisionar recursos que **não são nativamente suportados** pelo CloudFormation ou executar **lógica de provisionamento personalizada** durante as fases de criação, atualização e exclusão da Stack.

* **Tipo:** O tipo de recurso é declarado como **Custom::NomeDoRecurso**.  
* **Mecanismo de *Backend*:** O Custom Resource é suportado por:  
  * **AWS Lambda Function (Mais Comum):** A função Lambda contém a lógica de provisionamento.  
  * **Amazon SNS Topic:** Um tópico SNS.  
* **ServiceToken:** A propriedade ServiceToken aponta para o ARN da função Lambda ou do tópico SNS que executará a lógica.  
* **Caso de Uso Principal:** Executar um script que **esvazia um bucket S3** antes que o CloudFormation tente excluí-lo, resolvendo o problema de falha de exclusão de buckets não vazios. O Lambda é invocado pelo Custom Resource na fase de exclusão da Stack.

---

## **17\. StackSets**

StackSets são uma extensão do CloudFormation que permite gerenciar a implantação de uma Stack (e seus recursos) em **múltiplas Contas AWS** e **múltiplas Regiões AWS** a partir de uma **única operação**.

* **Administração Centralizada:** Um template é usado para criar um **StackSet** a partir de uma conta administrativa.  
* **Implantação em Escala:** O StackSet, por sua vez, cria **instâncias de Stack** nos destinos configurados (contas-alvo e regiões-alvo).  
* **AWS Organizations:** Um caso de uso comum é implantar infraestrutura padrão em **todos os membros** de uma AWS Organization, exigindo que a conta administrativa tenha as permissões necessárias para gerenciar as implantações.

---

## **Apêndice: Estrutura de um Template YAML**

Os templates de CloudFormation geralmente utilizam o formato **YAML** devido à sua maior legibilidade em comparação com o JSON.

### ** Blocos de Construção do Template**

O template é composto por diferentes seções (blocos), sendo **Resources** a única seção obrigatória:

| Seção | Descrição | Exemplo de Uso |
| :---- | :---- | :---- |
| **AWSTemplateFormatVersion** | Define a versão interna de como o template deve ser lido (uso interno da AWS). | Não essencial para o usuário final. |
| **Description** | Comentários e descrição sobre o propósito do template. | Description: My first EC2 instance. |
| **Resources** | **OBRIGATÓRIO.** Define todos os recursos da AWS que serão criados e seus tipos (ex: AWS::EC2::Instance). | MyInstance: Type: AWS::EC2::Instance |
| **Parameters** | **Inputs dinâmicos** para o template. Valores que o usuário deve fornecer no momento da criação ou atualização da Stack. | Pode ser usado para definir a descrição de um Security Group em tempo de execução. |
| **Mappings** | **Variáveis estáticas** para o template. | Usado tipicamente para associar IDs de AMI específicos a regiões. |
| **Outputs** | Define os valores de referência (ex: o ID público de uma instância) que podem ser facilmente recuperados ou usados por outras Stacks. | Expor o Public IP de uma EC2. |
| **Conditions** | Uma lista de condições para determinar se um recurso deve ser criado ou se uma propriedade deve ser usada. | Ex: Criar um banco de dados apenas se o ambiente for "Production". |
### ** Sintaxe Básica do YAML**

* **Pares Chave-Valor:** Documentos YAML são compostos por chaves seguidas de dois pontos (:) e seus respectivos valores.  
  ```  
  Key: Value
  ```

* **Objetos Aninhados (Hierarquia):** O aninhamento e a estrutura de objetos são definidos por **indentação** (espaços), não por chaves ({}) ou colchetes (\[\]).  
  ```  
  Recurso:  
    Propriedade1: Valor1  
    Propriedade2:  
      SubPropriedade: Valor2
  ```

* **Arrays (Listas):** Representados por um **traço/hífen** (-) seguido por um espaço.  
  ```  
  SecurityGroups:  
    - SecurityGroup1ID  
    - SecurityGroup2ID
  ```
* **Comentários:** Linhas que começam com o sinal de **hash** (\#) são ignoradas.
