---
title: "Microsoft Entra ID (옛 Azure AD)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:35:00+09:00
tags:
  - azure
  - security
  - entra-id
  - azure-ad
---

# Microsoft Entra ID

**[[security|↑ Security]]** · **[[../cloud-azure|↑↑ Azure]]**

---

## 1. 한 줄

Microsoft 의 **identity service** — 회사 SSO 의 표준. Office 365 / Azure / Windows 통합.

옛 이름 = Azure Active Directory (Azure AD). 2023 rebrand.

---

## 2. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Tenant** | 조직 (1 organization) |
| **User** | 사람 |
| **Group** | User 묶음 |
| **Service Principal** | 응용 / 자동화의 ID |
| **Managed Identity** | Azure 자원의 자동 SP (key 없음) |
| **App Registration** | OAuth / OIDC 응용 등록 |
| **Role (RBAC)** | Azure 자원 권한 |
| **Conditional Access** | 조건부 정책 (MFA / location / device) |

---

## 3. AWS / GCP 와 차이

```
AWS IAM        = AWS 자원 권한 + 일부 federated
GCP IAM        = GCP 자원 권한
Entra ID       = identity 자체 (사람 + 응용) — 광범위
Azure RBAC     = Azure 자원 권한 (Entra ID identity 사용)
```

→ Entra ID 는 Azure 만이 아니라 회사 전체 SSO.

---

## 4. Managed Identity

VM / Function / App Service 등에 자동 SP:

```hcl
resource "azurerm_linux_virtual_machine" "vm" {
  identity { type = "SystemAssigned" }
}

resource "azurerm_role_assignment" "blob_read" {
  scope                = azurerm_storage_account.main.id
  role_definition_name = "Storage Blob Data Reader"
  principal_id         = azurerm_linux_virtual_machine.vm.identity[0].principal_id
}
```

→ key 없이 응용이 Storage / Key Vault / SQL 사용.

---

## 5. SDK / Auth

```python
from azure.identity import DefaultAzureCredential
credential = DefaultAzureCredential()
# 자동: Managed Identity → CLI → env vars → VS Code

from azure.storage.blob import BlobServiceClient
client = BlobServiceClient("https://myappstorage.blob.core.windows.net", credential=credential)
```

---

## 6. RBAC

```bash
az role assignment create \
  --assignee alice@example.com \
  --role "Storage Blob Data Reader" \
  --scope /subscriptions/.../resourceGroups/myapp
```

| 기본 Role | 의미 |
| --- | --- |
| Owner | 권한 부여 가능 |
| Contributor | 자원 관리 (권한 X) |
| Reader | 읽기 |
| User Access Administrator | 권한 관리만 |

→ 수백 built-in role + custom.

---

## 7. Conditional Access

```
조건: user / app / location / device / sign-in risk
액션: Grant + MFA / Block / Session control
```

예 — 회사 외부 IP 에서 admin login = MFA 강제.

---

## 8. App Registration / OAuth

```bash
az ad app create --display-name "MyApp"
az ad sp create --id <app-id>
```

→ 응용을 Entra ID 의 OAuth client 로 등록. Frontend / Backend / API 인증.

---

## 9. Federation / SSO

- SAML / OIDC — 외부 SSO (Okta / Google / GitHub OIDC)
- Hybrid — on-prem AD ↔ Entra ID 동기 (Azure AD Connect)
- B2B — guest user (다른 tenant)
- B2C — 외부 사용자 (Cognito 동등)

---

## 10. 함정

- 옛 "Azure AD" / 새 "Entra ID" 두 이름 — 같은 것
- Service Principal 직접 만들기 보다 Managed Identity 권장
- key rotation 어려움 → MI / cert 권장
- Conditional Access 잘못 = lockout (emergency account 보존)
- Privileged Identity Management (PIM) 으로 admin 일시 활성

---

## 11. 관련

- [[security]]
- [[key-vault]] — secret
- [[../compute/vm]] — managed identity
