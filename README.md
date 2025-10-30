# AWS_Review_Certification_Developer
## Agrupa resumos em Markdown sobre os recursos da AWS que são objetivos da prova de certificação

# 🌐 Introdução ao Elastic Load Balancing (ELB) da AWS

### 1. O Que é Balanceamento de Carga?
* **Conceito:** Um servidor ou conjunto de servidores que **encaminha o tráfego recebido** para várias instâncias (servidores de back-end/downstream).
* **Mecanismo:** Distribui a carga entre as instâncias de forma equitativa (ou baseada em algoritmos).
* **Ponto Único de Acesso:** Usuários se conectam a um **ponto de extremidade (endpoint)** único do balanceador de carga, sem saber a qual instância de back-end estão conectados.

### 2. Benefícios Chave do Uso de um Load Balancer
* **Disponibilidade e Tolerância a Falhas:** Lida perfeitamente com falhas de instâncias de downstream (removendo-as do serviço).
* **Escalabilidade:** Distribui a carga, permitindo lidar com mais usuários.
* **Segurança:** Permite a terminação SSL/TLS (descarregando a criptografia) e separação de tráfego público e privado.
* **Visibilidade:** Oferece alta visibilidade entre zonas de disponibilidade.
* **Aderência (Sticky Sessions):** Possibilidade de reforçar a aderência com *cookies* (enviando o mesmo usuário sempre para a mesma instância).

### 3. Elastic Load Balancing (ELB) - Serviço Gerenciado
* **Natureza:** É um **balanceador de carga gerenciado** pela AWS.
* **Vantagens do Gerenciamento:**
    * A AWS garante o funcionamento, atualizações, manutenção e **alta disponibilidade**.
    * Geralmente, **custa menos e é menos complexo** do que configurar e gerenciar seu próprio balanceador.
* **Integrações:** Integrado com muitos outros serviços AWS (EC2, Auto Scaling Groups, IAM, CloudWatch, Route 53, etc.).

### 4. Verificações de Saúde (Health Checks)
* **Propósito:** Mecanismo crucial para o ELB verificar se uma instância de back-end está **funcionando corretamente**.
* **Como Funciona:**
    * É feito através de um **protocolo**, uma **porta** e uma **rota** específica (Ex: HTTP, Porta 4567, Rota `/health`).
    * Se a instância **não responder com uma resposta OK** (geralmente código `200`), ela é marcada como **"Não Saudável"** e o ELB para de enviar tráfego para ela.

### 5. Tipos de Balanceadores de Carga Gerenciados (Gerações)

| Tipo | Geração | Ano | Camadas/Protocolos Suportados | Recomendação |
| :--- | :--- | :--- | :--- | :--- |
| **Classic Load Balancer (CLB)** | V1 | 2009 | HTTP, HTTPS, TCP, SSL | **Obsoleto.** AWS recomenda usar as gerações mais recentes. |
| **Application Load Balancer (ALB)** | V2 | 2016 | **HTTP, HTTPS**, WebSocket | Ótimo para aplicações web e microsserviços. |
| **Network Load Balancer (NLB)** | V3 | 2017 | **TCP, TLS, UDP** | Ideal para tráfego de alta performance e ultra-baixa latência. |
| **Gateway Load Balancer (GLB)** | V4 | 2020 | **Camada de Rede (IP)** | Usado para inspeção de tráfego e integração com dispositivos de rede de terceiros. |

* **Tipos de Acesso:** Podem ser configurados como **Internos** (privado para a VPC) ou **Externos/Públicos** (para acesso de sites e aplicações públicos).

### 6. Segurança (Grupos de Segurança)
* **Regra do ALB:** O Security Group do ALB geralmente permite tráfego na Porta 80 (HTTP) e/ou 443 (HTTPS) de **Qualquer lugar** (`0.0.0.0/0`).
* **Regra das Instâncias EC2 (Mecanismo Aprimorado):**
    * O Security Group das instâncias de back-end deve permitir o tráfego apenas na Porta 80.
    * A **Origem** do tráfego deve ser vinculada ao **Security Group do próprio Load Balancer**.
    * **Resultado:** A instância só permite tráfego se ele for originado do Load Balancer, adicionando uma camada extra de segurança.

## 🌐 ALB - Application Load Balancer (ALB) da AWS

### 1. Conceito Fundamental
* **Camada de Operação:** Camada 7 (Aplicação - HTTP).
* **Função:** Permite direcionar para **vários aplicativos HTTP** em máquinas.
* **Organização:** As máquinas são agrupadas em **Grupos-Alvo**.
* **Balanceamento Multi-App:** Permite fazer balanceamento de carga para múltiplos aplicativos na **mesma instância do EC2** (ideal para contêineres/ECS).
* **Vantagem sobre CLB (Classic Load Balancer):** Um único ALB pode balancear vários aplicativos, enquanto o CLB exigiria um balanceador por aplicativo.

### 2. Recursos de Protocolo e Roteamento
O ALB oferece um roteamento inteligente baseado em regras:

#### A. Suporte a Protocolos
* Suporte a **HTTP/2** e **WebSockets**.
* **Redirecionamento:** Suporta o redirecionamento automático de **HTTP para HTTPS** no nível do balanceador de carga.

#### B. Roteamento Avançado (Baseado em Conteúdo)
O roteamento pode ser feito para diferentes Grupos-Alvo com base em:

| Tipo de Roteamento | Exemplo |
| :--- | :--- |
| **Caminho da URL (Path)** | Roteia `/users` para o Grupo A e `/posts` para o Grupo B. |
| **Nome do Host** | Roteia `a.exemplo.com` para o Grupo C e `b.exemplo.com` para o Grupo D. |
| **Strings de Consulta e Cabeçalhos** | Roteia `exemplo.com/reservas?id=123&order=false` para um Grupo E. |

### 3. Grupos-Alvo (Target Groups)
Os Grupos-Alvo são os destinos reais para onde o tráfego é roteado.
* **Destinos Suportados:**
    * Instâncias EC2 (podem ser gerenciadas pelo **Grupo de Auto Scaling**).
    * Tarefas ECS (Containers).
    * Funções **Lambda** (base do Serverless).
    * Endereços **IP Privados** (permite balanceamento para servidores on-premises/data center privado).
* **Monitoramento:** As **Verificações de Saúde (Health Checks)** são realizadas no nível do Grupo-Alvo.

### 4. Detalhes Técnicos sobre a Conexão
* **Nome Fixo:** Você obtém um nome de host fixo (como no CLB).
* **Encerramento de Conexão:** O ALB é quem se comunica diretamente com o cliente e faz o encerramento da conexão.
* **Visibilidade do IP do Cliente:** O servidor de aplicação (EC2) não vê o IP do cliente diretamente, mas sim o IP privado do ALB.
    * O IP real do cliente, a porta e o protocolo são inseridos em cabeçalhos HTTP extras:
        * `X-Forwarded-For` (IP do cliente)
        * `X-Forwarded-Ports` (Porta do cliente)
        * `X-Forwarded-Proto` (Protocolo usado)

# ⚡ Network Load Balancer (NLB) da AWS

### 1. Conceito e Camada de Operação
* **Camada de Operação:** Camada 4 (Transporte).
* **Protocolos Suportados:** **TCP** e **UDP**.
    * *Dica para o Exame:* Ao ver "UDP" ou alto desempenho TCP, pense em NLB.

### 2. Características de Desempenho e Endereçamento
* **Desempenho Extremo:** Projetado para lidar com **milhões de solicitações por segundo**.
* **Latência:** Apresenta **latência ultrabaixa**.
* **Endereçamento IP Estático:**
    * Tem um **IP estático por Zona de Disponibilidade (AZ)**.
    * Você pode atribuir um **IP Elástico (EIP)** a cada AZ.
    * *Dica para o Exame:* Se o seu aplicativo precisa ser acessado por um conjunto **fixo/estático** de IPs, o NLB é a opção ideal.

### 3. Funcionamento e Grupos-Alvo (Target Groups)
O NLB encaminha o tráfego para **Grupos-Alvo** que podem ser:

* **Instâncias EC2:** O NLB envia tráfego TCP ou UDP diretamente para as instâncias.
* **Endereços IP Privados:**
    * É possível registrar IPs privados de instâncias EC2 ou de **servidores no seu próprio Data Center (on-premises)**, permitindo o balanceamento de tráfego híbrido.
* **Combinação NLB + ALB:**
    * É possível ter um **NLB na frente de um ALB** (Network Load Balancer antes de um Application Load Balancer).
    * **Motivo:** O NLB fornece os **endereços IP fixos**, e o ALB fornece as **regras avançadas de roteamento** baseadas em HTTP/Camada 7.

### 4. Verificações de Saúde (Health Checks)
As verificações de integridade nos Grupos-Alvo do NLB suportam três protocolos diferentes: **TCP** | **HTTP** | **HTTPS**
