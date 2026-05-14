---
title: "OCI Vault"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:25:00+09:00
tags:
  - oci
  - security
  - vault
  - kms
  - secret
---

# OCI Vault

**[[security|↑ Security]]** · **[[../cloud-oci|↑↑ OCI]]**

---

## 1. 한 줄

KMS + Secret Manager — Key Vault / KMS+Secrets Manager 동등. **Virtual Private Vault** 옵션 (FIPS HSM).

---

## 2. 사용

```hcl
resource "oci_kms_vault" "vault" {
  compartment_id = var.compartment_ocid
  display_name   = "myapp-vault"
  vault_type     = "DEFAULT"             # 또는 VIRTUAL_PRIVATE
}

resource "oci_kms_key" "data" {
  compartment_id      = var.compartment_ocid
  display_name        = "data-key"
  management_endpoint = oci_kms_vault.vault.management_endpoint

  key_shape {
    algorithm = "AES"
    length    = 32                       # 256 bit
  }

  protection_mode = "HSM"                # 또는 SOFTWARE
}

resource "oci_vault_secret" "db_password" {
  compartment_id = var.compartment_ocid
  vault_id       = oci_kms_vault.vault.id
  key_id         = oci_kms_key.data.id
  secret_name    = "db-password"

  secret_content {
    content_type = "BASE64"
    content      = base64encode(var.db_password)
  }
}
```

---

## 3. SDK

```python
import oci
signer = oci.auth.signers.InstancePrincipalsSecurityTokenSigner()
client = oci.secrets.SecretsClient(config={}, signer=signer)

bundle = client.get_secret_bundle(secret_id="ocid1.vaultsecret.oc1...")
secret = base64.b64decode(bundle.data.secret_bundle_content.content).decode()
```

→ instance principal 로 자동.

---

## 4. Key 사용

- Object Storage / Block Volume / DB encryption — CMK 지정
- Envelope encryption — data key 자체는 OCI 가 wrap

---

## 5. Rotation / Versions

```
secret-version
  - CURRENT
  - PREVIOUS
  - PENDING
  - DEPRECATED
```

→ rotation 후 응용이 PREVIOUS 사용 가능 (rollback).

---

## 6. 가격

```
Default vault:        무료
Virtual Private:      $$$
Key:                  $0.0185 / month (Software) or $1 / month (HSM)
Operations:           대부분 무료
Secret:               $0.40 / month / secret
```

---

## 7. 함정

- Default Vault 의 throughput 한계 — high QPS = VPV
- HSM key 삭제 = 30+ 일 grace
- secret rotation 자동 X — 응용 / Function 으로
- region 별 별도 vault
- policy 의 secret-family / vault-family

---

## 8. 관련

- [[security]]
- [[iam]]
- [[../storage/object-storage]]
- [[../database/autonomous-db]]
