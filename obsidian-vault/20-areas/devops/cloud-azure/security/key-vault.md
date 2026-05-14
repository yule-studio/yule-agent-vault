---
title: "Azure Key Vault"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:40:00+09:00
tags:
  - azure
  - security
  - key-vault
---

# Azure Key Vault

**[[security|↑ Security]]** · **[[../cloud-azure|↑↑ Azure]]**

---

## 1. 한 줄

KMS + Secret Manager + Certificate Manager 통합. AWS KMS + Secrets Manager + ACM 의 1 서비스.

---

## 2. 종류

| 항목 | 의미 |
| --- | --- |
| **Key** | 암호화 / 서명 key (HSM 옵션) |
| **Secret** | password / API key |
| **Certificate** | TLS 인증서 + auto-renewal |

---

## 3. 사용

```bash
az keyvault create -g myapp -n myapp-kv -l koreacentral \
  --enable-rbac-authorization true \
  --enable-purge-protection true

az keyvault secret set --vault-name myapp-kv --name db-password --value "..."
az keyvault secret show --vault-name myapp-kv --name db-password
```

```hcl
resource "azurerm_key_vault" "kv" {
  name                       = "myapp-kv"
  location                   = "koreacentral"
  resource_group_name        = azurerm_resource_group.main.name
  tenant_id                  = data.azurerm_client_config.current.tenant_id
  sku_name                   = "standard"        # 또는 premium (HSM)
  enable_rbac_authorization  = true
  purge_protection_enabled   = true
  soft_delete_retention_days = 30

  network_acls {
    default_action = "Deny"
    bypass         = "AzureServices"
    ip_rules       = ["..."]
  }
}

resource "azurerm_key_vault_secret" "db" {
  name         = "db-password"
  value        = var.password
  key_vault_id = azurerm_key_vault.kv.id
}

resource "azurerm_role_assignment" "app_read" {
  scope                = azurerm_key_vault.kv.id
  role_definition_name = "Key Vault Secrets User"
  principal_id         = azurerm_linux_virtual_machine.vm.identity[0].principal_id
}
```

---

## 4. SDK

```python
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

client = SecretClient(vault_url="https://myapp-kv.vault.azure.net", credential=DefaultAzureCredential())
secret = client.get_secret("db-password")
print(secret.value)
```

→ Managed Identity 통합.

---

## 5. App Service / Function 통합

```hcl
app_settings = {
  "DB_PASSWORD" = "@Microsoft.KeyVault(SecretUri=https://myapp-kv.vault.azure.net/secrets/db-password)"
}
```

→ 환경변수에 KV 참조. 응용 코드 무관.

---

## 6. Certificate (TLS)

```bash
az keyvault certificate create --vault-name myapp-kv -n mycert \
  --policy "$(az keyvault certificate get-default-policy)"
```

자동 renewal — Let's Encrypt 통합 (옛, 변경 가능). App Gateway / Front Door 가 KV cert 직접 사용.

---

## 7. HSM (Premium)

- FIPS 140-2 Level 3
- BYOK (Bring Your Own Key)
- Managed HSM = dedicated

---

## 8. 비용

```
Standard:
  $0.03 / 10K operation
  Certificate: $3 / cert / month
Premium (HSM):
  $1 / key / month + operations
Managed HSM:
  $3.20 / HSM·시간 (~$2300/월)
```

---

## 9. 함정

- soft delete 활성 (default 90일) — 삭제 후 복구
- purge protection — 영구 보호 (production 필수)
- network ACL 잘못 = lockout
- key vault per tenant — cross-tenant 어려움
- API rate limit (vault 별)

---

## 10. 관련

- [[security]]
- [[entra-id]] — auth
- [[../database/azure-sql]] / [[../compute/vm]] — 사용
