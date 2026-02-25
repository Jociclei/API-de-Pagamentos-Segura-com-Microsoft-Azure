# 💳 API de Pagamentos Segura com Microsoft Azure
> Projeto desenvolvido como parte do desafio prático da DIO — Microsoft Application Platform

---

## 📋 Descrição do Projeto

Desenvolvimento de uma **API de Pagamentos** robusta e segura utilizando serviços da **Microsoft Azure**, com foco em autenticação, autorização, proteção de dados sensíveis e rastreabilidade completa de transações financeiras.

---

## 🏗️ Arquitetura da Solução

```
┌──────────────────────────────────────────────────────────────┐
│                    CLIENTES DA API                           │
│          (App Mobile / E-Commerce / Parceiros)               │
└────────────────────────┬─────────────────────────────────────┘
                         │ HTTPS + OAuth 2.0
              ┌──────────▼──────────┐
              │  Azure API Management│
              │  • Rate Limiting     │
              │  • Autenticação JWT  │
              │  • Throttling        │
              │  • Logs de Auditoria │
              └──────────┬──────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
  ┌──────▼──────┐ ┌──────▼──────┐ ┌─────▼──────┐
  │   Payments  │ │  Webhook    │ │  Fraud     │
  │     API     │ │  Service    │ │  Detection │
  │ (App Svc)   │ │ (Functions) │ │ (Functions)│
  └──────┬──────┘ └─────────────┘ └────────────┘
         │
┌────────▼────────────────────────────────────┐
│              DADOS & SEGURANÇA               │
│  ┌──────────────┐  ┌───────────────────────┐ │
│  │  Azure SQL   │  │    Azure Key Vault    │ │
│  │  Database    │  │  (Secrets, Certs,     │ │
│  │(Transações)  │  │   API Keys terceiros) │ │
│  └──────────────┘  └───────────────────────┘ │
│  ┌──────────────┐  ┌───────────────────────┐ │
│  │  Azure AD B2C│  │   Azure Service Bus   │ │
│  │  (Identidade │  │  (Fila de Pagamentos) │ │
│  │  de usuários)│  │                       │ │
│  └──────────────┘  └───────────────────────┘ │
└─────────────────────────────────────────────┘
         │
┌────────▼────────────────────┐
│     MONITORAMENTO            │
│  Azure Monitor + Sentinel    │
│  Application Insights        │
│  (Alertas de fraude/anomalia)│
└─────────────────────────────┘
```

---

## ☁️ Serviços Azure Utilizados

| Serviço | Finalidade |
|---|---|
| **Azure API Management** | Gateway central: autenticação, rate limiting, auditoria |
| **Azure App Service** | Hospedagem da API de pagamentos |
| **Azure AD B2C** | Identidade e autenticação de usuários/clientes |
| **Azure Key Vault** | Armazenamento seguro de secrets, chaves e certificados |
| **Azure SQL Database** | Persistência das transações financeiras |
| **Azure Service Bus** | Fila assíncrona para processamento de pagamentos |
| **Azure Functions** | Webhooks, notificações e detecção de fraudes |
| **Azure Monitor + Sentinel** | Monitoramento, alertas e detecção de ameaças |
| **Application Insights** | Rastreamento de performance e erros |
| **Azure Private Endpoint** | Acesso privado ao banco sem expor à internet |

---

## 🔒 Estratégia de Segurança (Camadas)

```
Camada 1 — PERÍMETRO
  └── Azure API Management: JWT validation, IP allowlist, rate limit

Camada 2 — IDENTIDADE
  └── Azure AD B2C: OAuth 2.0 + OIDC, MFA obrigatório para admin

Camada 3 — DADOS EM TRÂNSITO
  └── TLS 1.3 obrigatório, certificados gerenciados pelo Azure

Camada 4 — DADOS EM REPOUSO
  └── Azure SQL TDE (Transparent Data Encryption)
  └── Key Vault para chaves de criptografia (CMK)

Camada 5 — SEGREDOS E CREDENCIAIS
  └── Key Vault: zero secrets no código ou variáveis de ambiente
  └── Managed Identity para acesso entre serviços

Camada 6 — AUDITORIA
  └── Todos os endpoints logados no Azure Monitor
  └── Alertas automáticos para padrões suspeitos (Azure Sentinel)
```

---

## 🚀 Passo a Passo da Implementação

### 1. Resource Group e Identidade
```bash
# Criar Resource Group
az group create \
  --name rg-payments-api-dio \
  --location brazilsouth

# Criar Managed Identity para a API
az identity create \
  --name id-payments-api \
  --resource-group rg-payments-api-dio
```

### 2. Azure Key Vault — Cofre de Segredos
```bash
az keyvault create \
  --name kv-payments-dio \
  --resource-group rg-payments-api-dio \
  --location brazilsouth \
  --sku standard \
  --enable-soft-delete true \
  --retention-days 90

# Armazenar secrets dos gateways de pagamento
az keyvault secret set \
  --vault-name kv-payments-dio \
  --name "stripe-secret-key" \
  --value "sk_live_xxxxx"

az keyvault secret set \
  --vault-name kv-payments-dio \
  --name "database-connection-string" \
  --value "Server=sql-payments-dio.database.windows.net;..."

# Dar acesso da API ao Key Vault via Managed Identity
az keyvault set-policy \
  --name kv-payments-dio \
  --object-id $(az identity show --name id-payments-api \
    --resource-group rg-payments-api-dio --query principalId -o tsv) \
  --secret-permissions get list
```

### 3. Azure SQL Database — Transações Financeiras
```bash
az sql server create \
  --name sql-payments-dio \
  --resource-group rg-payments-api-dio \
  --location brazilsouth \
  --admin-user payadmin \
  --admin-password "PaySecure@2024!"

# Habilitar Advanced Threat Protection
az sql server advanced-threat-protection-setting update \
  --resource-group rg-payments-api-dio \
  --server-name sql-payments-dio \
  --state Enabled

az sql db create \
  --resource-group rg-payments-api-dio \
  --server sql-payments-dio \
  --name db-payments \
  --service-objective S2 \
  --zone-redundant false
```

### 4. Schema do Banco de Dados
```sql
-- Tabela de transações (PCI-DSS inspired)
CREATE TABLE Transacoes (
    id                UNIQUEIDENTIFIER  PRIMARY KEY DEFAULT NEWID(),
    external_id       NVARCHAR(100)     NOT NULL UNIQUE,  -- ID do gateway externo
    usuario_id        NVARCHAR(100)     NOT NULL,
    valor             DECIMAL(12,2)     NOT NULL,
    moeda             CHAR(3)           NOT NULL DEFAULT 'BRL',
    status            NVARCHAR(50)      NOT NULL,  -- pending|processing|approved|failed|refunded
    metodo_pagamento  NVARCHAR(50)      NOT NULL,  -- credit_card|pix|boleto
    gateway           NVARCHAR(50)      NOT NULL,  -- stripe|pagseguro|mercadopago
    -- Dados de cartão NUNCA armazenados em texto puro
    cartao_ultimos4   CHAR(4),
    cartao_bandeira   NVARCHAR(20),
    ip_origem         NVARCHAR(50),
    idempotency_key   NVARCHAR(100)     UNIQUE,    -- Previne duplicatas
    created_at        DATETIME2         DEFAULT GETDATE(),
    updated_at        DATETIME2         DEFAULT GETDATE()
);

-- Tabela de auditoria (append-only, nunca atualizada)
CREATE TABLE AuditLog (
    id             BIGINT IDENTITY(1,1) PRIMARY KEY,
    transacao_id   UNIQUEIDENTIFIER  REFERENCES Transacoes(id),
    evento         NVARCHAR(100)     NOT NULL,
    detalhes       NVARCHAR(MAX),
    usuario        NVARCHAR(100),
    ip             NVARCHAR(50),
    timestamp      DATETIME2         DEFAULT GETDATE()
);

-- Índices para performance
CREATE INDEX idx_transacoes_usuario ON Transacoes(usuario_id, created_at);
CREATE INDEX idx_transacoes_status  ON Transacoes(status, created_at);
```

### 5. Azure API Management — Gateway Central
```bash
az apim create \
  --name apim-payments-dio \
  --resource-group rg-payments-api-dio \
  --location brazilsouth \
  --publisher-email "admin@empresa.com" \
  --publisher-name "Payments DIO" \
  --sku-name Developer
```

**Política de segurança no APIM (XML):**
```xml
<policies>
  <inbound>
    <!-- Validar JWT do Azure AD B2C -->
    <validate-jwt header-name="Authorization" failed-validation-httpcode="401">
      <openid-config url="https://login.microsoftonline.com/{tenant}/v2.0/.well-known/openid-configuration"/>
      <required-claims>
        <claim name="scope" match="any">
          <value>payments.write</value>
        </claim>
      </required-claims>
    </validate-jwt>

    <!-- Rate limiting: 100 chamadas/minuto por usuário -->
    <rate-limit-by-key calls="100" renewal-period="60"
      counter-key="@(context.Request.Headers.GetValueOrDefault("Authorization",""))" />

    <!-- Chave de idempotência obrigatória -->
    <check-header name="Idempotency-Key" failed-check-httpcode="400"
      failed-check-error-message="Idempotency-Key header is required" />
  </inbound>

  <backend>
    <forward-request />
  </backend>

  <outbound>
    <!-- Remover headers sensíveis da resposta -->
    <set-header name="X-Powered-By" exists-action="delete" />
    <set-header name="Server" exists-action="delete" />
  </outbound>
</policies>
```

### 6. Azure Service Bus — Fila de Pagamentos
```bash
az servicebus namespace create \
  --resource-group rg-payments-api-dio \
  --name sb-payments-dio \
  --location brazilsouth \
  --sku Standard

az servicebus queue create \
  --resource-group rg-payments-api-dio \
  --namespace-name sb-payments-dio \
  --name payment-processing \
  --lock-duration PT5M \
  --max-delivery-count 3 \
  --dead-lettering-on-message-expiration true
```

---

## 📡 Endpoints da API

```
POST   /api/v1/payments              → Criar novo pagamento
GET    /api/v1/payments/{id}         → Consultar status do pagamento
POST   /api/v1/payments/{id}/refund  → Solicitar reembolso
GET    /api/v1/payments/history      → Histórico de pagamentos do usuário
POST   /api/v1/webhooks/stripe       → Receber eventos do Stripe
```

**Exemplo de Request — Criar Pagamento:**
```json
POST /api/v1/payments
Authorization: Bearer {jwt_token}
Idempotency-Key: uuid-v4-unico

{
  "valor": 150.00,
  "moeda": "BRL",
  "metodo": "credit_card",
  "cartao": {
    "token": "tok_stripe_xxxx"
  },
  "descricao": "Pedido #12345"
}
```

**Exemplo de Response:**
```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "status": "processing",
  "valor": 150.00,
  "moeda": "BRL",
  "created_at": "2024-01-15T10:30:00Z",
  "_links": {
    "self": "/api/v1/payments/3fa85f64-...",
    "status": "/api/v1/payments/3fa85f64-.../status"
  }
}
```

---

## 💡 Insights e Aprendizados

### Idempotência é Crítica em Pagamentos
Redes falham. Clientes clicam "Pagar" duas vezes. Sem idempotência, o cliente pode ser cobrado em duplicata. A solução foi exigir um `Idempotency-Key` UUID único em cada request de pagamento. O APIM e o banco garantem que a mesma chave nunca processa dois pagamentos.

### Key Vault + Managed Identity = Segurança Sem Atrito
Antes eu imaginava que segurança era complexa. Com Managed Identity, a API acessa o Key Vault sem nenhuma senha no código. A Azure gerencia os tokens de acesso automaticamente — e se a aplicação for comprometida, o atacante não tem acesso às credenciais.

### Azure Service Bus Desacopla e Protege
Processar pagamentos de forma síncrona é arriscado — timeouts causam estados inconsistentes. Com o Service Bus, a API responde imediatamente "processing" e uma Azure Function processa o pagamento na fila. Se o processador de pagamento estiver fora do ar, as mensagens ficam na fila com retry automático.

### APIM como Primeira Linha de Defesa
Centralizar autenticação, rate limiting e validação de headers no API Management mantém a API limpa e focada em regras de negócio. Bloqueios por IP suspeito e throttling acontecem antes de qualquer código da aplicação executar.

---

## 🔮 Possibilidades de Evolução

- **Azure Fraud Protection** — detecção de fraudes em tempo real com ML da Microsoft
- **Azure Confidential Computing** — processamento de dados de cartão em ambiente TEE (Trusted Execution Environment)
- **PCI-DSS Compliance** — usar Azure como ambiente certificado PCI-DSS para reduzir escopo de auditoria
- **Multi-região ativo-ativo** — Azure Traffic Manager + SQL Geo-Replication para disponibilidade global
- **Azure Event Grid** — notificações em tempo real para sistemas downstream ao confirmar pagamento

---

## 💰 Estimativa de Custos

| Serviço | Tier | Custo/mês |
|---|---|---|
| Azure API Management | Developer | ~$50 USD |
| App Service | B2 | ~$75 USD |
| Azure SQL | S2 (50 DTUs) | ~$75 USD |
| Key Vault | Standard | ~$5 USD |
| Service Bus | Standard | ~$10 USD |
| Azure AD B2C | 50k MAU grátis | $0 USD |
| Application Insights | Pay-per-use | ~$5-15 USD |
| **Total Estimado** | | **~$220 USD/mês** |

---

## 🛠️ Tecnologias

![Azure](https://img.shields.io/badge/Microsoft_Azure-0089D6?style=for-the-badge&logo=microsoft-azure&logoColor=white)
![.NET](https://img.shields.io/badge/.NET-512BD4?style=for-the-badge&logo=dotnet&logoColor=white)
![SQL](https://img.shields.io/badge/Azure_SQL-CC2927?style=for-the-badge&logo=microsoft-sql-server&logoColor=white)
![Key Vault](https://img.shields.io/badge/Key_Vault-0078D4?style=for-the-badge&logo=microsoft-azure&logoColor=white)
![Service Bus](https://img.shields.io/badge/Service_Bus-0078D4?style=for-the-badge&logo=microsoft-azure&logoColor=white)

---

## 📚 Referências

- [Azure API Management Docs](https://docs.microsoft.com/azure/api-management)
- [Azure Key Vault Best Practices](https://docs.microsoft.com/azure/key-vault/general/best-practices)
- [Repositório Base DIO](https://github.com/digitalinnovationone/Microsoft_Application_Platform)
- [PCI-DSS no Azure](https://docs.microsoft.com/azure/compliance/offerings/offering-pci-dss)

---

*⭐ Se este projeto foi útil, deixe uma estrela no repositório!*
