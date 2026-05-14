---
title: "OCI IAM + Compartment"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:20:00+09:00
tags:
  - oci
  - security
  - iam
  - compartment
---

# OCI IAM

**[[security|↑ Security]]** · **[[../cloud-oci|↑↑ OCI]]**

---

## 1. 한 줄

OCI 의 IAM — **Compartment** 가 격리 단위, **Policy** 는 사람-읽기 친화 DSL.

---

## 2. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Tenancy** | 조직 (root compartment) |
| **Compartment** | 논리 격리 — 트리 구조 |
| **User** | 사람 |
| **Group** | User 묶음 |
| **Dynamic Group** | 자원 그룹 (matching rule) |
| **Policy** | 사람-읽기 DSL — group + verb + resource + condition |
| **Identity Domain** | identity scope (옛 Cloud Native vs Federated) |
| **Federation** | Microsoft Entra / Okta SSO |

---

## 3. Compartment

```
tenancy (root)
  ├── prod
  │     ├── network
  │     ├── compute
  │     └── data
  ├── dev
  └── sandbox
```

→ 자원이 어느 compartment 에 있는지 = 비용 / 권한 / 거버넌스 단위.

---

## 4. Policy DSL

```
Allow group <group> to <verb> <resource-type> in compartment <name> [where <condition>]
```

예:
```
Allow group DevOps to manage all-resources in compartment prod

Allow group AppDev to read objects in compartment prod
                  where target.bucket.name = 'public-data'

Allow group Auditor to inspect all-resources in tenancy

Allow dynamic-group AppInstances to read secret-bundles in compartment prod
                  where target.secret.id = 'ocid1.vaultsecret.oc1...'
```

| Verb | 의미 |
| --- | --- |
| inspect | 메타만 |
| read | 읽기 |
| use | 사용 (start/stop) |
| manage | 모두 |

---

## 5. Instance Principal (Dynamic Group)

```hcl
resource "oci_identity_dynamic_group" "vm_group" {
  name = "AppInstances"
  matching_rule = "ALL {instance.compartment.id = '${var.compartment_ocid}'}"
}

resource "oci_identity_policy" "vm_policy" {
  statements = [
    "Allow dynamic-group AppInstances to read objects in compartment prod",
    "Allow dynamic-group AppInstances to read secret-bundles in compartment prod"
  ]
}
```

→ VM 안 SDK 자동 인증.

```python
import oci
signer = oci.auth.signers.InstancePrincipalsSecurityTokenSigner()
client = oci.object_storage.ObjectStorageClient(config={}, signer=signer)
```

---

## 6. Federation

- **Identity Domain** = OCI 안의 identity 묶음 (옛 IDCS)
- 외부 SSO — Azure Entra / Okta / ADFS / Google Workspace
- B2B / B2C

---

## 7. MFA / Compartment Quotas

- MFA 강제 (모든 admin)
- Compartment Quota — region / service / compartment 별 한도

```
set network-load-balancer quota network-load-balancer-count to 5 in compartment dev
```

---

## 8. 함정

- policy 의 영향 범위 — compartment 가 sub-tree 전체
- root compartment 는 tenancy administrator 만
- instance principal vs resource principal (Functions)
- federation token 만료 후 reauth
- 옛 IDCS 와 새 Identity Domain — 마이그레이션

---

## 9. 관련

- [[security]]
- [[vault]]
- [[../cloud-oci|↑↑ OCI]]
