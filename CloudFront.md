# **🌐 AWS CloudFront**

## 1. Conceito e Objetivo Principal

O AWS CloudFront é a **Rede de Distribuição de Conteúdo (CDN)** da Amazon Web Services. CloudFront melhora o **desempenho de leitura** e a **experiência do usuário** ao distribuir e armazenar em cache o conteúdo do seu site globalmente.

---

## 2. **⚙️ Como Funciona**

1. **Rede Global:** É composto por **centenas de Pontos de Presença (PoP)** e caches de borda distribuídos em todo o mundo (locais de borda).  
2. **Baixa Latência:** Quando um usuário solicita um conteúdo, ele é atendido pelo **Local de Borda mais próximo**.  
3. **Mecanismo de Cache:** O Local de Borda serve o conteúdo diretamente (*cache hit*) ou busca na Origem (*cache miss*), armazena e serve.

---

## 3. **📈 Monitoramento e Análise**

### **Logs em Tempo Real (Real-Time Logs)**

Permite enviar todas as solicitações recebidas pelo CloudFront para um serviço de *streaming* para monitoramento, análise e ação em tempo real sobre o desempenho da entrega de conteúdo.

* **Destino:** O CloudFront envia os logs diretamente para um **Amazon Kinesis Data Stream**.  
* **Processamento:**  
  * Para processamento **quase em tempo real**, o fluxo Kinesis pode ser consumido por funções **AWS Lambda**.  
  * Para processamento em **lotes** (batch) e envio para destinos como S3 ou OpenSearch, o **Kinesis Data Firehose** é a ferramenta indicada.  
* **Controle:** É possível configurar:  
  * **Taxa de Amostragem:** A porcentagem de solicitações que devem ser enviadas ao Kinesis (útil para endpoints de alto tráfego).  
  * **Campos de Log:** Quais campos de dados da solicitação devem ser incluídos.  
  * **Comportamentos de Cache Específicos:** Permite filtrar os logs apenas por determinados padrões de caminho (/caminho/\*) ou comportamentos de cache.

---

## 4. **💰 Opções de Preços e Otimização de Custo**

| Classe de Preço | Descrição | Cobertura Geográfica | Custo Relativo |
| :---- | :---- | :---- | :---- |
| **Price Class All** | Inclui **todas** as regiões globais. | Melhor desempenho, cobertura mundial. | Mais Alto |
| **Price Class 200** | Inclui a **maioria** das regiões. | Exclui as regiões PoP de custo mais elevado. | Médio |
| **Price Class 100** | Inclui apenas as regiões **menos caras**. | América do Norte e Europa. | Mais Baixo |

---

## 5. **💾 Caching e Otimização Avançada**

### **🔑 Chave de Cache (Cache Key)**

* É o **identificador exclusivo** para o objeto em cache.

### **🔄 Invalidações de Cache**

* Força a **remoção imediata** do conteúdo do cache em todos os locais de borda (ignora o TTL).

### **🧭 Comportamentos de Cache (Cache Behaviors) e Múltiplas Origens**

* Permite a definição de diferentes regras de cache, TTLs e **Origens** com base em **padrões de caminho da URL**.

### **🔄 Grupos de Origem (Origin Groups) \- Alta Disponibilidade**

* **Finalidade:** Fornecer **Failover** e **Alta Disponibilidade** usando uma Origem Primária e uma Secundária.  
* **Caso de Uso DR:** Usado com buckets S3 replicados em regiões diferentes para obter recuperação de desastres.

---

## 6. **🛡️ Benefícios de Segurança e Conformidade**

* **Proteção contra DDoS** e integração com **AWS Shield** e **WAF**.  
* **Restrição Geográfica (Geo Restriction):** Restringe o acesso com base no país de origem do usuário.

### **🔒 URLs e Cookies Assinados (Signed URLs & Cookies)**

* Usados para fornecer acesso seguro a **conteúdo premium ou privado** por meio da distribuição do CloudFront.

### **🔐 Criptografia em Nível de Campo (Field-Level Encryption)**

* **Finalidade:** Proteger informações sensíveis (ex: cartão de crédito) em **toda a pilha** de aplicação.  
* **Mecanismo:** O Ponto de Presença **criptografa** campos específicos na solicitação POST usando uma **Chave Pública**. Apenas o servidor de origem, com a Chave Privada, pode descriptografar.

---

## 7. **🏠 Tipos de Origem (Backend)**

| Tipo de Origem | Descrição | Segurança/Conexão | Notas Importantes |
| :---- | :---- | :---- | :---- |
| **Amazon S3** | Buckets do S3. | Protegido por **OAC (Origin Access Control)**. | Ideal para conteúdo estático. |
| **Origem VPC** | Aplicativos em sub-redes **privadas** (ALB, EC2, NLB). | **Conexão privada direta** (melhor e mais segura). | O backend não é exposto publicamente na internet. |
| **Origem Personalizada** | Qualquer backend **HTTP** (dentro ou fora da AWS). | Deve ser um backend HTTP público. | Inclui o **Método Antigo (Público Restrito)** de conexão via IPs do CloudFront. |

---

## 8. **🆚 CloudFront vs. Replicação entre Regiões S3**

| Característica | AWS CloudFront (CDN) | Replicação entre Regiões S3 |
| :---- | :---- | :---- |
| **Escopo** | Rede Global Edge (Pontos de Presença mundiais). | Configurado para regiões específicas. |
| **Atualização** | Armazena em **cache** (duração definida). | Quase em **tempo real** (sem cache). |
| **Uso Ideal** | Conteúdo **Estático** distribuído globalmente com baixa latência. | Conteúdo **Dinâmico** que precisa mudar constantemente em regiões específicas. |

