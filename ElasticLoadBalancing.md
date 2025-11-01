# 🌐 Introdução ao Elastic Load Balancing (ELB) da AWS

## 1. Conceito e Objetivo Principal
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

# 🛡️ 4. Gateway Load Balancer (GLB) da AWS

### 4.1. Propósito e Casos de Uso
* **Função Principal:** Implantar, dimensionar e gerenciar uma frota de **dispositivos virtuais de rede de terceiros** na AWS.
* **Uso:** É essencialmente um *loop* na rede para forçar o tráfego a passar por:
    * **Firewalls** (de terceiros).
    * Sistemas de **Detecção/Prevenção de Intrusão (IDS/IPS)**.
    * Sistemas de **Inspeção Profunda de Pacotes (DPI)**.
    * **Dispositivos de modificação de carga útil** no nível da rede.

### 4.2. Características e Funcionamento
* **Camada de Operação:** Camada 3 (Rede - Pacotes IP). É o nível mais baixo de operação entre os Load Balancers.
* **Protocolo de Comunicação:** Usa o **Protocolo GENEVE** na porta `6081` para trocar tráfego com os dispositivos virtuais de destino.
* **Funcionalidade Dupla:**
    1.  **Gateway de Rede Transparente:** Todo o tráfego da VPC é roteado para ele através de modificações nas **Tabelas de Rota**, tornando sua inspeção transparente para a aplicação.
    2.  **Balanceador de Carga:** Distribui o tráfego entre a frota de dispositivos virtuais no Grupo-Alvo.

### 4.3. Fluxo de Tráfego (Conceito Chave)
1.  O tráfego dos usuários é interceptado pelas tabelas de rota e enviado ao **GLB**.
2.  O GLB distribui o tráfego para o Grupo-Alvo de **Dispositivos Virtuais (Appliances)**.
3.  Os dispositivos inspecionam (ou modificam) o tráfego.
4.  Se aceito, o tráfego é enviado de volta ao **GLB**.
5.  O GLB encaminha o tráfego inspecionado para o **Aplicativo** (de forma transparente).

### 4.4. Grupos-Alvo (Target Groups)
* **Destinos Suportados:**
    * Instâncias EC2 (onde rodam os *appliances* de rede).
    * Endereços **IP Privados** (útil se você estiver rodando dispositivos virtuais em seu próprio data center).

# 📍 5. Sessões Fixas (Sticky Sessions) / Afinidade de Sessão

### 5.1. Conceito e Propósito
* **Definição:** É a capacidade de garantir que um **cliente** que faz múltiplas solicitações ao balanceador de carga seja **sempre direcionado para a mesma instância de back-end**.
* **Caso de Uso:**
    * Principalmente usado para **manter os dados da sessão** (ex: carrinho de compras, status de login) em uma única instância de backend.
* **Comportamento Padrão vs. Aderência:**
    * **Padrão:** O ELB distribui a carga de todas as solicitações uniformemente.
    * **Aderência:** Desvia do padrão, enviando solicitações subsequentes do mesmo cliente para a instância inicial.
* **Desvantagem:** Pode **desequilibrar a carga** entre as instâncias se um único usuário ou um grupo de usuários gerar um tráfego muito alto ("usuários muito aderentes").

### 5.2. Habilitação e Escopo
* **Serviços Suportados:** Pode ser ativado para **Classic Load Balancer (CLB)**, **Application Load Balancer (ALB)** e **Network Load Balancer (NLB)**.
* **Nível de Configuração:** A aderência é configurada no nível do **Grupo-Alvo (Target Group)**.
* **Mecanismo:** É implementado usando **cookies** que são enviados entre o balanceador de carga e o cliente. A aderência termina quando o cookie expira.

### 5.3. Tipos de Cookies para Aderência

Existem dois tipos principais de cookies usados para implementar a aderência:

| Tipo de Cookie | Geração | Propriedades | Nome Padrão do Cookie (ALB) |
| :--- | :--- | :--- | :--- |
| **Baseado em Duração** | Gerado pelo **Balanceador de Carga (ELB)**. | Possui uma expiração baseada em uma **duração específica** (ex: 1 segundo a 7 dias) definida no ELB. | `AWSALB` (para ALB) ou `AWSELB` (para CLB). |
| **Baseado em Aplicativo** | Gerado pelo **Destino (Instância / Aplicativo)**. | É um cookie **personalizado** que pode incluir atributos definidos pelo aplicativo. O ELB apenas usa o nome do cookie para aderência. | `AWSALBAPP` (para ALB). |

* **Nomes Reservados:** Não devem ser usados nomes de cookies reservados pela AWS: `AWSALB`, `AWSALBAPP`, `AWSALBTG`.

### 5.4. Como Funciona (Fluxo Básico)
1.  **Primeira Solicitação:** Cliente faz a solicitação ao ALB; o ALB a encaminha para a Instância A.
2.  **Resposta:** O ALB envia uma resposta ao cliente que **inclui o cookie de aderência** (com a informação de roteamento e data de expiração).
3.  **Solicitações Subsequentes:** O navegador do cliente envia o **cookie de aderência** em todas as solicitações seguintes.
4.  **Roteamento:** O ALB lê o cookie e roteia a solicitação diretamente para a **Instância A**.

# 🗺️ 6. Balanceamento de Zonas Cruzadas (Cross-Zone Load Balancing - CZLB)

### 6.1. Conceito
* **Definição:** Mecanismo que permite que uma instância do balanceador de carga em uma Zona de Disponibilidade (AZ) distribua o tráfego uniformemente para **todas as instâncias de destino** registradas, **em todas as AZs**, e não apenas para as instâncias dentro de sua própria AZ.

### 6.2. Comportamentos de Roteamento

| Cenário | Descrição | Impacto no Tráfego |
| :--- | :--- | :--- |
| **Com CZLB Ativado** (Exemplo: 2 instâncias no AZ-A, 8 instâncias no AZ-B = 10 no total) | Cada nó do Load Balancer distribui seu tráfego para **TODAS as 10 instâncias**. | O tráfego é distribuído **uniformemente** entre todas as instâncias EC2, independentemente da AZ. Cada instância recebe 10% do tráfego total. |
| **Sem CZLB (Padrão ou Desativado)** | Cada nó do Load Balancer distribui o tráfego **apenas para as instâncias em sua própria AZ**. | Se o número de instâncias for desequilibrado, as instâncias em AZs com menos targets receberão **mais tráfego**. |

> **Exemplo Sem CZLB:** O AZ-A (com 2 instâncias) receberá 50% do tráfego total. Cada instância do AZ-A receberá 25% do tráfego total (50% / 2).

### 6.3. Status de Cobrança e Padrão por Tipo de Load Balancer

| Tipo de Load Balancer | Padrão CZLB | Cobrança por Transferência de Dados Inter-AZ |
| :--- | :--- | :--- |
| **Application Load Balancer (ALB)** | **LIGADO (Ativado)** | **NÃO COBRADO** (mesmo que haja tráfego entre AZs). |
| **Network Load Balancer (NLB)** | **DESLIGADO (Desativado)** | **COBRADO** se for ativado. |
| **Gateway Load Balancer (GLB)** | **DESLIGADO (Desativado)** | **COBRADO** se for ativado. |
| **Classic Load Balancer (CLB)** | DESLIGADO | **NÃO COBRADO** se for ativado. |

* **Observação de Cobrança:** A AWS normalmente cobra por transferências de dados entre AZs. O **ALB** é a **exceção notável**, onde o CZLB é gratuito.

### 6.4. Configuração (Onde Habilitar/Desabilitar)
* **ALB:** O CZLB está **SEMPRE LIGADO** no nível do balanceador. Para desativá-lo, é preciso fazê-lo no nível do **Grupo-Alvo** (Target Group).
* **NLB & GLB:** O CZLB é **DESLIGADO** por padrão e pode ser ativado nos **atributos do Load Balancer**.

# 🔒 7. Certificados SSL/TLS e SNI (Server Name Indication)

### 7.1. Conceitos Fundamentais
* **Propósito:** Permite que o tráfego entre os clientes e o balanceador de carga seja **criptografado em trânsito** (criptografia em voo).
* **Terminologia:**
    * **SSL (Secure Sockets Layer):** Termo mais antigo, ainda amplamente usado.
    * **TLS (Transport Layer Security):** Versão mais recente e tecnicamente correta (mais utilizada hoje).
* **Validade:** Certificados têm data de validade e devem ser renovados regularmente.
* **Autoridades Certificadoras (CAs):** Empresas que emitem certificados SSL públicos (ex: Comodo, GoDaddy, Letsencrypt).

### 7.2. Terminação SSL/TLS na AWS
* **Local:** Os certificados SSL (públicos ou personalizados) são carregados no **Balanceador de Carga (ELB)**.
* **Processo:** O ELB realiza a **Terminação de Certificado SSL/TLS** (descriptografa o tráfego do cliente).
* **Comunicação Back-end:**
    * O tráfego do cliente para o ELB é **HTTPS** (criptografado).
    * O tráfego do ELB para a instância EC2 pode ser **HTTP** (não criptografado), pois o tráfego está contido dentro da rede privada da VPC.
* **Gerenciamento de Certificados:** Os certificados são gerenciados usando o **AWS Certificate Manager (ACM)**, onde você pode solicitar ou carregar seus próprios certificados (formato X.509).

### 7.3. Server Name Indication (SNI)
* **Problema que Resolve:** Permite carregar **múltiplos certificados SSL em um único balanceador de carga/servidor web** para atender a vários sites/domínios.
* **Mecanismo:** O cliente deve indicar o **nome do host de destino** durante o handshake SSL inicial. O ELB usa essa informação para carregar o certificado correto para o domínio solicitado.

### 7.4. Compatibilidade com Certificados (SNI)

O suporte a múltiplos certificados (via SNI) varia entre os tipos de Load Balancers:

| Tipo de Load Balancer | Suporte SNI (Múltiplos Certificados) | Padrão |
| :--- | :--- | :--- |
| **Application Load Balancer (ALB)** | **SIM** | Suporta vários ouvintes e certificados (v2). |
| **Network Load Balancer (NLB)** | **SIM** | Suporta vários ouvintes e certificados (v2). |
| **Classic Load Balancer (CLB)** | **NÃO** | Suporta apenas **UM** certificado SSL. Para múltiplos domínios, seriam necessários múltiplos CLBs.

# 💧 8. Drenagem de Conexão (Connection Draining) / Atraso de Cancelamento (Deregistration Delay)

### 8.1. Conceito e Terminologia
* **Propósito:** Conceder tempo suficiente para que uma instância de destino conclua as **solicitações ativas/em andamento** antes de ser retirada do serviço (desregistrada ou marcada como não íntegra).
* **Nomes Diferentes:**
    * **Classic Load Balancer (CLB):** Chamado de **Drenagem de Conexão (Connection Draining)**.
    * **ALB/NLB/GLB (Gerações mais novas):** Chamado de **Atraso de Cancelamento (Deregistration Delay)**.

### 8.2. Fluxo de Operação (Referência ao Diagrama)
O diagrama ilustra o estado de "Drenagem":

1.  **Instância em Drenagem:** Quando uma instância EC2 (superior) está sendo desregistrada ou marcada como não íntegra, o ELB a coloca no estado **DRAINING**.
2.  **Tráfego Existente:** O ELB permite que as **conexões existentes** (dos usuários já conectados) a essa instância se **concluam** ("waiting for existing connections to complete").
3.  **Novas Conexões:** O ELB **para de enviar novas solicitações** a essa instância.
4.  **Roteamento:** O ELB estabelece novas conexões **apenas com as outras instâncias saudáveis** (inferiores).
5.  **Término:** Após a conclusão de todas as solicitações em andamento (ou após o tempo limite), a instância é retirada de serviço.

### 8.3. Configuração
* **Local de Configuração:** O tempo de drenagem é parametrizável.
* **Valor Padrão:** **300 segundos (5 minutos)**.
* **Intervalo:** Pode ser configurado entre 1 e **3600 segundos (1 hora)**.
* **Desativação:** Definir o valor como **zero (0)** desativa o recurso (sem drenagem).

### 8.4. Implicações na Aplicação
| Cenário da Aplicação | Configuração Recomendada | Vantagem/Desvantagem |
| :--- | :--- | :--- |
| **Solicitações Curtas** (Ex: < 1 segundo) | Valor baixo (Ex: 30 segundos) | A instância sai de linha rapidamente. |
| **Solicitações Longas** (Ex: Uploads, Long-Polling) | Valor alto (Ex: 300+ segundos) | Garante que as solicitações longas não sejam perdidas. A desvantagem é que a instância demora mais para sair de linha.
