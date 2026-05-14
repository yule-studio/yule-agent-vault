---
title: "GCP Cloud KMS"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:40:00+09:00
tags:
  - gcp
  - security
  - kms
---

# GCP Cloud KMS

**[[security|↑ Security]]** · **[[../cloud-gcp|↑↑ GCP]]**

---

## 1. 한 줄

암호화 key 매니지드 — AES / RSA / EC. 모든 GCP 암호화 (GCS / PD / SQL / Secret) 의 backing.

---

## 2. 구조

```
KeyRing (region / global)
  └── Key (CryptoKey)
        └── Version (active / disabled / destroyed)
```

---

## 3. 생성

```bash
gcloud kms keyrings create myapp --location=asia-northeast3

gcloud kms keys create db-key \
  --keyring=myapp --location=asia-northeast3 \
  --purpose=encryption \
  --rotation-period=90d \
  --next-rotation-time=2026-08-15T00:00:00Z
```

```hcl
resource "google_kms_key_ring" "myapp" {
  name     = "myapp"
  location = "asia-northeast3"
}

resource "google_kms_crypto_key" "db" {
  name            = "db-key"
  key_ring        = google_kms_key_ring.myapp.id
  rotation_period = "7776000s"            # 90일

  lifecycle { prevent_destroy = true }
}
```

---

## 4. 사용

```python
from google.cloud import kms
client = kms.KeyManagementServiceClient()
key_name = client.crypto_key_path("PROJECT","asia-northeast3","myapp","db-key")

resp = client.encrypt(request={"name": key_name, "plaintext": b"secret"})
ciphertext = resp.ciphertext

resp = client.decrypt(request={"name": key_name, "ciphertext": ciphertext})
plaintext = resp.plaintext
```

→ envelope encryption — KMS 가 DEK 생성, 응용 AES.

---

## 5. CMEK (Customer-Managed Encryption Key)

```hcl
resource "google_sql_database_instance" "myapp" {
  settings {
    disk_encryption_configuration {
      kms_key_name = google_kms_crypto_key.db.id
    }
  }
}
```

→ GCS / PD / Cloud SQL / BigQuery 등에 명시 — Google managed key 대신 사용자 key.

---

## 6. 비용

```
$0.06 / key·월 (active version)
$0.03 / 10K operation
Asymmetric: 더 비쌈
HSM: $1 / key·월
```

---

## 7. 함정

- destroy 후 30일 recovery (default). 짧게 / 길게 조정.
- key rotation 시 옛 version 도 유지 (decrypt 가능)
- region binding — multi-region key 가능
- AWS KMS 와 같은 격리 모델
- audit log 의 Data Access 활성 필요 (event 보려면)

---

## 8. 관련

- [[security]]
- [[secret-manager]]
- [[../storage/cloud-storage]] / [[../database/cloud-sql]] — CMEK
