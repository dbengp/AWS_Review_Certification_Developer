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
