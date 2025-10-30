# 📈 AWS Auto Scaling Group (ASG)

## 1. Conceito e Objetivo Principal
* **Definição:** É um serviço que gerencia uma coleção de instâncias EC2, automatizando a criação e remoção de servidores em resposta à demanda.
* **Objetivo:** Garantir que o número de instâncias EC2 em execução corresponda à carga do aplicativo.
* **Escalabilidade Horizontal:**
    * **Scale Out (Aumentar a escala):** Adicionar instâncias EC2 para lidar com carga aumentada.
    * **Scale In (Diminuir a escala):** Remover instâncias EC2 para lidar com carga reduzida.
* **Custo:** O serviço ASG é **gratuito**. Você paga apenas pelos recursos (instâncias EC2, volumes EBS, etc.) que ele provisiona.

## 2. Parâmetros de Capacidade
O ASG exige que você defina limites para gerenciar a escala:

| Parâmetro | Função | Exemplo |
| :--- | :--- | :--- |
| **Capacidade Mínima (Min Capacity)** | O menor número de instâncias que deve estar em execução no ASG em qualquer momento. | 2 |
| **Capacidade Desejada (Desired Capacity)** | O número atual de instâncias que você deseja que o ASG mantenha. | 4 |
| **Capacidade Máxima (Max Capacity)** | O maior número de instâncias permitido no ASG. | 7 |

## 3. Superpoderes e Alta Disponibilidade
O ASG é fundamental para a resiliência e a integração de serviços:

* **Substituição de Instâncias:** Se uma instância for marcada como **não íntegra** (Unhealthy), o ASG a **encerra** e **cria uma nova instância** para substituí-la, mantendo a capacidade desejada.
* **Integração com ELB:**
    * Todas as instâncias criadas pelo ASG são automaticamente **vinculadas (registradas)** ao Load Balancer.
    * O ASG pode usar as **Verificações de Saúde (Health Checks)** do ELB para determinar se deve encerrar uma instância não íntegra.

## 4. Atributos para Criação (Modelo de Execução)
Para criar um ASG, é necessário um **Modelo de Execução (Launch Template)**, que substituiu as antigas Configurações de Inicialização. O modelo contém todas as informações necessárias para lançar uma instância EC2:

* **AMI** e **Tipo de Instância** (t2.micro, m5.large, etc.).
* **Dados do Usuário (User Data)** (scripts de inicialização).
* **Volumes EBS** e **Grupos de Segurança**.
* **Par de Chaves SSH** e **Funções IAM**.
* **Informações de Rede** (Sub-redes) e **Configurações do Load Balancer**.

## 5. Políticas de Dimensionamento (Scaling Policies)
O recurso **"Automático"** do ASG é alcançado através da integração com o **CloudWatch** e Políticas de Dimensionamento.

* **Alarme do CloudWatch:** Um alarme é acionado quando uma **métrica** específica (ex: **CPU Média**, Tráfego de Rede, Métrica Personalizada) atinge um limite definido.
* **Políticas de Scale Out:** Acionadas quando a carga aumenta (Ex: CPU Média > 70%), resultando no **aumento** do número de instâncias.
* **Políticas de Scale In:** Acionadas quando a carga diminui (Ex: CPU Média < 30%), resultando na **remoção** de instâncias (down-sizing).

### 5.1. Tipos de Políticas
Existem quatro tipos principais de políticas que definem como o ASG deve alterar sua capacidade:
| Tipo de Política | Descrição | Casos de Uso |
| :--- | :--- | :--- |
| **Rastreamento de Metas (Target Tracking)** | **Mais simples de configurar.** Defina uma **métrica** (ex: Utilização da CPU) e um **valor-alvo** (ex: 40%). O ASG dimensiona-se automaticamente para manter a média da métrica próxima desse alvo. | Manter a utilização média da CPU em um nível ideal. |
| **Simples/Por Etapas (Simple/Step Scaling)** | Baseia-se em **Alarmes do CloudWatch**. Define ações específicas (adicionar X unidades, remover Y unidades) quando um alarme é acionado. | Ações mais granulares ou não lineares em resposta a picos. |
| **Programado (Scheduled Scaling)** | Dimensionamento com base em um **padrão de uso conhecido e antecipado**. A capacidade é ajustada em horários predefinidos. | Ex: Aumentar a capacidade mínima toda Sexta-feira às 17h, antes do pico do fim de semana. |
| **Preditivo (Predictive Scaling)** | Analisa a **carga histórica** e padrões que se repetem (dados cíclicos). Gera uma previsão e **agenda ações de dimensionamento** com antecedência para se preparar para a carga futura. | Padrões de tráfego que se repetem diariamente ou semanalmente (mais avançado). |

### 5.2. Métricas de Dimensionamento Recomendadas
Escolher a métrica correta é crucial para um dimensionamento eficiente. Boas métricas incluem:

* **Utilização da CPU:** Métrica padrão e eficaz, pois a maioria das solicitações utiliza capacidade de processamento.
* **RequestCountPerTarget (Contagem de Solicitações por Destino):** Útil ao usar um Load Balancer. Você define um alvo de requisições por instância (ex: 1000 requisições/destino) e o ASG escala para manter esse alvo.
* **Média de Entrada/Saída da Rede (Network In/Out):** Importante se o gargalo for a largura de banda (ex: muitas operações de upload ou download).
* **Métricas Personalizadas:** Qualquer métrica específica do aplicativo enviada ao CloudWatch pode ser usada.

### 5.3. Período de Estabilização (Cooldown Period)
* **Definição:** Um período de espera obrigatório após uma atividade de dimensionamento (adicionar ou remover instâncias).
* **Padrão:** **5 minutos (300 segundos)**.
* **Função:** Impede que o ASG inicie ou encerre instâncias adicionais imediatamente. Isso permite que as **métricas se estabilizem** e as novas instâncias entrem em vigor antes de tomar a próxima decisão de dimensionamento.
* **Otimização:** Usar uma **AMI pré-configurada** (pronta para uso) reduz o tempo de inicialização da instância, permitindo que o Cooldown seja reduzido, resultando em um **dimensionamento mais dinâmico**.

