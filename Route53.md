# **Amazon Route 53**

## 1. Conceito e Objetivo Principal
O Amazon Route 53 é um serviço DNS **altamente disponível, escalável, totalmente gerenciado e autoritativo** da AWS. Seu nome faz referência à porta DNS tradicional (53).
* **Registrador de Domínio:** Permite comprar e registrar nomes de domínio.  
* **DNS Autoritativo:** É a fonte de verdade para os registros do seu domínio.  
* **Verificação de Saúde (Health Checks):** Permite rotear o tráfego apenas para recursos saudáveis.  
* **Alta Disponibilidade:** É o **único** serviço AWS que oferece um **SLA de 100% de disponibilidade**.

#### **Tipos de Registros DNS Cruciais para o Exame:**

| Tipo de Registro | Função | Restrições Importantes |
| :---- | :---- | :---- |
| **A** | Mapeia um nome de *host* para um endereço **IPv4**. | Não há. |
| **AAAA** | Mapeia um nome de *host* para um endereço **IPv6**. | Não há. |
| **CNAME (Canonical Name)** | Mapeia um nome de *host* para **outro nome de *host***. | **Não pode ser usado no *Apex*** (a raiz do domínio, ex: exemplo.com). |
| **NS (Name Server)** | Indica os servidores DNS que são autoritativos para a Zona Hospedada. | Criado automaticamente. |

#### **Zonas Hospedadas (Hosted Zones): Pública vs. Privada**

As zonas hospedadas definem como o tráfego é roteado para um domínio e seus subdomínios.

| Característica | Zona Hospedada Pública | Zona Hospedada Privada |
| :---- | :---- | :---- |
| **Acessibilidade** | Consultável por **qualquer pessoa** na Internet. | Consultável **apenas** a partir de recursos dentro da sua **Amazon VPC**. |
| **Uso** | Domínios públicos (ex: app.exemplo.com). | Nomes de domínio internos (ex: database.exemplo.internal). |
| **Custo** | $0,50 USD por mês. | $0,50 USD por mês. |

---

### **3\. Prática: Configuração e Verificação**

#### **Registro e Criação de Registros Simples**

1. **Registro de Domínio:** Pague a taxa anual (mínimo $12) e registre seu domínio. Ative a **Proteção de Privacidade** para ocultar suas informações de contato.  
2. **Criação da Zona:** Ao registrar o domínio, o Route 53 cria automaticamente uma **Zona Hospedada Pública** com os registros iniciais **NS** e **SOA**.  
3. **Criação de um Registro A:** Para mapear um subdomínio (teste.stefanetheteacher.com) para um IP, você cria um **Registro A**, define o **TTL (Time To Live)** (o tempo que a resposta ficará em cache) e a política de roteamento (por enquanto, **Simple Routing**).  
4. **Verificação:** A resolução de DNS pode ser verificada usando ferramentas de linha de comando:  
   * nslookup \[domínio\]  
   * dig \[domínio\] (preferido por mostrar mais detalhes, como o TTL e o tipo de registro).

#### **Preparação da Infraestrutura (Para Roteamento Avançado)**

Para testar as políticas de roteamento do Route 53, é necessário ter *endpoints* reais em diferentes localizações geográficas e um Application Load Balancer (ALB).

1. **Implantação de EC2:** Três instâncias EC2 são criadas em regiões distintas (ex: UE Central 1, US East 1, AP Southeast 1), cada uma executando um servidor web simples que retorna um "Hello World" e a **Availability Zone (AZ)** em que se encontra, para identificação.  
2. **Criação de um ALB:** Um Application Load Balancer é provisionado (neste exemplo, em Frankfurt) e configurado com um Grupo Alvo (Target Group) apontando para a instância EC2 local (em Frankfurt).  
3. **Validação:** Acessar os IPs públicos de cada EC2 e o Nome DNS do ALB confirma que todos os *endpoints* de aplicação estão funcionais e prontos para serem usados como destino nos registros do Route 53\.

### **4\. Time To Live (TTL) – Tempo de Vida**

O **Time To Live (TTL)** é um valor (em segundos) que a AWS define em cada registro DNS para instruir os resolvedores de DNS e clientes (navegadores) a **armazenar em cache** a resposta da consulta.

| Característica | TTL Alto (Ex: 24 horas) | TTL Baixo (Ex: 60 segundos) |
| :---- | :---- | :---- |
| **Tráfego DNS (Custo)** | **Menos** tráfego (consultas), resultando em **custo menor** no Route 53\. | **Mais** tráfego (consultas), resultando em **custo maior**. |
| **Atualização do Registro** | **Lenta.** Demora mais para todos os clientes verem a nova alteração, pois o valor antigo fica em cache por mais tempo. | **Rápida.** O tempo de inatividade com um registro antigo é minimizado. |
| **Estratégia Comum** | Para alterar um registro com TTL alto: 1\. Diminua o TTL (Ex: para 60s). 2\. Espere o TTL antigo expirar. 3\. Altere o registro (a mudança entra em vigor rapidamente). 4\. Aumente o TTL novamente. | Recomendado para implantações ou durante manutenção, quando mudanças são esperadas. |
| **Obrigatoriedade** | É obrigatório para **todos os tipos de registro**, exceto o **Alias Record** (onde o TTL é definido automaticamente pelo Route 53). |  |

### **5\. CNAME vs. Alias Record (Crucial para o Exame)**

O Route 53 oferece duas maneiras de mapear um nome de *host* para outro, mas o **Alias** é a solução nativa da AWS e a preferida para a maioria dos recursos.

| Característica | CNAME (Canonical Name) | Alias Record (Registro Alias) |
| :---- | :---- | :---- |
| **Alvo** | Qualquer **Nome de Host** (DNS Name) em qualquer lugar (AWS ou fora). | Apenas **Recursos AWS Específicos** (ALB, CloudFront, S3 Website, API Gateway, etc.). |
| **Apex Domain** | **NÃO PERMITIDO** no *Apex* (raiz do domínio, ex: meudominio.com). | **PERMITIDO** no *Apex* (resolve a restrição do CNAME). |
| **TTL** | **Obrigatório** (você define o valor). | **Não pode ser definido** (TTL gerenciado e otimizado automaticamente pelo Route 53). |
| **Custo de Consulta** | **Cobrado** por cada consulta DNS. | **Gratuito** (sem custo de consulta). |
| **Resolução de IP** | O cliente (ou o resolvedor) deve fazer uma **consulta extra** para resolver o IP final. | **Resolve o IP automaticamente** no lado do Route 53; reconhece mudanças de IP do alvo. |
| **Alvos Não Suportados** | **EC2 DNS Names** não são alvos válidos para Alias Records. |  |

💡 **Ponto de Certificação:** Para rotear o domínio raiz (meudominio.com) para um recurso AWS (como um ALB), você **DEVE** usar um **Alias Record**.

### **6\. Políticas de Roteamento do Route 53**

As políticas de roteamento determinam como o Route 53 **responde** a uma consulta DNS. **Importante:** O DNS não roteia o tráfego em si; ele apenas fornece o *endpoint* (IP) para onde o cliente deve ir.

| Política de Roteamento | Objetivo e Funcionamento | Caso de Uso |
| :---- | :---- | :---- |
| **Simple (Simples)** | Roteia o tráfego para um único recurso. Permite **múltiplos valores** no mesmo registro; o cliente escolhe um IP **aleatoriamente**. | Simples (ex: um único servidor). **Não** pode ser associado a **Health Checks**. |
| **Weighted (Ponderada)** | Roteia o tráfego para múltiplos recursos com base em um **peso** relativo que você define. A porcentagem de tráfego é proporcional ao peso. | **Load Balancing entre regiões** (o ALB não faz isso), **Testes Canário** (enviar 10% do tráfego para uma nova versão). |
| **Latency-Based (Baseada em Latência)** | Roteia o tráfego para o *endpoint* que oferece a **menor latência** (tempo de resposta mais rápido) para o usuário que faz a consulta. | Aplicações de **multirregião** onde o desempenho (velocidade) é a principal preocupação. **Requer que você especifique a Região AWS** do *endpoint*. |
| **Failover (Ativo/Passivo)** | **Foco em Disponibilidade.** Roteia o tráfego para um recurso **Primário**. Se o *Health Check* do primário falhar, o Route 53 move automaticamente o tráfego para o recurso **Secundário** (Disaster Recovery). | **Configuração de Disaster Recovery** (ativo/passivo). O recurso primário **DEVE** ter um *Health Check* associado. |
| **Geolocation (Localização Geográfica)** | Roteia o tráfego com base na **localização geográfica** do usuário (continente, país, ou estado/província). | Exibir conteúdo específico da região, garantir a conformidade com regras regionais de dados ou distribuir a carga globalmente. **Requer um registro *Default*** para usuários que não correspondem a nenhuma regra. |
| **Geoproximity (Proximidade Geográfica)** | Roteia o tráfego com base na **distância geográfica** entre usuários e recursos. Permite usar **Bias (Viés)**. | **Load Balancing Complexo:** O **Bias** (positivo para atrair mais tráfego, negativo para repelir) permite "empurrar" ou "puxar" o tráfego de/para uma região, alterando a linha divisória geográfica. |
| **Multi-Value Answer (Resposta de Múltiplos Valores)** | Retorna até **8 registros saudáveis** aleatoriamente. **Permite Health Checks** para garantir que os IPs retornados estejam funcionais. | Roteamento básico com alta disponibilidade de *endpoints*. Não substitui um ELB, mas oferece *load balancing* do lado do cliente. |
| **IP-Based (Baseada em IP)** | Roteia o tráfego com base em **CIDR Blocks** (intervalos de IP) definidos pelo cliente. | Otimizar desempenho ou reduzir custos para tráfego proveniente de **Provedores de Internet (ISP)** ou faixas de IP específicas. |

### **7\. Health Checks (Verificações de Saúde)**

O Route 53 permite verificar a saúde dos seus *endpoints* e garantir que o tráfego DNS seja roteado apenas para recursos funcionais.

#### **Tipos de Health Checks:**

1. **Monitorar um Endpoint (Público):**  
   * Verifica a saúde de um recurso público (servidor, ALB, aplicação) enviando solicitações de **cerca de 15 *checkers*** globais da AWS.  
   * **Intervalo:** Normal (30 segundos) ou Rápido (10 segundos, custo maior).  
   * **Saúde:** O *endpoint* é considerado **saudável** se retornar um código de *status* **2xx** ou **3xx** ou se um **texto específico** for encontrado nos primeiros 5.120 bytes da resposta.  
   * **Regra de Firewall:** Você deve permitir o tráfego de entrada dos **IPs dos *Health Checkers*** do Route 53 no seu *Security Group*.  
2. **Health Check Calculado (Calculated Health Check):**  
   * Combina o resultado de **vários *Health Checks*** (até 256 *checks* filhos) em um único *check* pai.  
   * **Condição:** A saúde do *check* pai é determinada por condições **OR**, **AND** ou **NOT** aplicadas aos *checks* filhos, ou exigindo que um número mínimo de *checks* filhos passe.  
   * **Uso:** Manutenção ou lógica complexa de *failover*.  
3. **Monitorar um Alarme do CloudWatch (Para Recursos Privados):**  
   * **Uso:** É a maneira **recomendada** de monitorar a saúde de recursos **privados** (dentro de uma VPC ou *on-premises*), pois os *Health Checkers* do Route 53 são públicos e não podem acessá-los.  
   * **Mecanismo:** Você cria uma **Métrica do CloudWatch** para o recurso privado (ex: Latência, Erros), cria um **Alarme do CloudWatch** que dispara quando o limite da métrica é violado e, em seguida, associa esse Alarme ao *Health Check* do Route 53\.  
   * **Resultado:** Quando o Alarme entra no estado ALARM, o *Health Check* do Route 53 fica **não saudável**.

### **8\. Distinção: Registrador de Domínio vs. Serviço DNS**

É crucial entender que as funções de **Registrar Domínio** e de **Gerenciar Registros DNS** são serviços distintos, embora frequentemente oferecidos pela mesma empresa (como o Route 53 ou GoDaddy).

* **Registrador de Domínio:** Onde você **compra e renova** anualmente o nome do seu domínio (ex: exemplo.com). Você paga as taxas anuais aqui.  
* **Serviço DNS:** Onde você **gerencia** os registros do seu domínio (A, CNAME, etc.) e onde sua **Zona Hospedada** reside.

#### **Cenário de Uso Misto (Importante para o Exame):**

Você pode registrar seu domínio com um terceiro (ex: **GoDaddy**), mas usar o **Amazon Route 53** para gerenciar seus registros DNS:

1. Crie uma **Zona Hospedada Pública** no Route 53 para o seu domínio.  
2. O Route 53 fornecerá 4 **Servidores de Nomes (Name Servers \- NS)**.  
3. Vá ao site do Registrador (GoDaddy) e **atualize os registros NS** do seu domínio para apontar para os 4 NS fornecidos pelo Route 53\.  
4. A partir desse momento, o **Route 53** se torna o **Serviço DNS autoritativo** para o seu domínio, e todas as consultas serão respondidas com base nos registros que você criar nele.

### **Apêndice: DNS**

O **DNS (Domain Name System)** é a espinha dorsal da Internet, responsável por traduzir nomes de *host* amigáveis (URLs) para os endereços IP numéricos dos servidores de destino.

#### **Terminologia Essencial:**

| Termo | Descrição |
| :---- | :---- |
| **Registrador de Domínios** | Entidade onde você registra seu nome de domínio (ex: Amazon Route 53, GoDaddy). |
| **Registros DNS** | Entradas que mapeiam nomes de *host* para IPs ou outros nomes (ex: A, CNAME, NS, AAAA). |
| **Zona Hospedada (Zone File)** | Um contêiner que armazena todos os registros DNS de um domínio ou subdomínio. |
| **Servidores de Nomes (Name Servers \- NS)** | Servidores que resolvem as consultas de DNS para uma zona. |
| **TLD (Top-Level Domain)** | O último nível do nome de domínio (ex: .com, .org, .br). |
| **Domínio de Segundo Nível** | O nome exclusivo logo antes do TLD (ex: amazon.com, google.com). |
| **FQDN (Fully Qualified Domain Name)** | O nome de domínio completo e absoluto (ex: api.www.exemplo.com). |

#### **O Mecanismo de Consulta do DNS (Processo Recursivo):**

Quando um navegador solicita um URL (exemplo.com), ocorre uma consulta recursiva:

1. O Navegador pergunta ao **Servidor DNS Local** (gerenciado pelo ISP ou empresa).  
2. Se não souber, o Servidor DNS Local pergunta ao **Servidor DNS Raiz** (root).  
3. O Servidor Raiz direciona para o **Servidor de Domínio de Nível Superior (TLD)** (ex: o servidor .com).  
4. O Servidor TLD direciona para o **Servidor de Nomes do Domínio** (autoritativo, que será o Route 53).  
5. O Servidor de Nomes do Domínio retorna o **endereço IP** (Registro A).  
6. O Servidor DNS Local **armazena a resposta em cache (TTL)** e a envia ao Navegador.  
7. O Navegador usa o IP para acessar o servidor web.
