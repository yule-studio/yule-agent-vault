---
title: "GCP Secret Manager"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:35:00+09:00
tags:
  - gcp
  - security
  - secrets
---

# GCP Secret Manager

**[[security|↑ Security]]** · **[[../cloud-gcp|↑↑ GCP]]**

---

## 1. 한 줄

DB password / API key / 인증서 의 매니지드 저장 + IAM access + KMS 암호화 + 버전 관리.

---

## 2. 사용

```bash
echo -n "supersecret" | gcloud secrets create db-password --data-file=- --replication-policy=automatic

gcloud secrets versions access latest --secret=db-password

# Update
echo -n "new" | gcloud secrets versions add db-password --data-file=-

# IAM
gcloud secrets add-iam-policy-binding db-password \
  --member=serviceAccount:myapp@PROJECT.iam.gserviceaccount.com \
  --role=roles/secretmanager.secretAccessor
```

```hcl
resource "google_secret_manager_secret" "db" {
  secret_id = "db-password"
  replication { auto {} }                         # 또는 user_managed (region 명시)
}

resource "google_secret_manager_secret_version" "db" {
  secret      = google_secret_manager_secret.db.id
  secret_data = var.db_password
}

resource "google_secret_manager_secret_iam_member" "app" {
  secret_id = google_secret_manager_secret.db.id
  role      = "roles/secretmanager.secretAccessor"
  member    = "serviceAccount:${google_service_account.app.email}"
}
```

---

## 3. SDK 사용

```python
from google.cloud import secretmanager
client = secretmanager.SecretManagerServiceClient()
resp = client.access_secret_version(name="projects/PROJECT/secrets/db-password/versions/latest")
password = resp.payload.data.decode("utf-8")
```

---

## 4. Cloud Run / GKE 통합

Cloud Run:
```hcl
template {
  containers {
    env {
      name = "DB_PASSWORD"
      value_source {
        secret_key_ref {
          secret  = google_secret_manager_secret.db.secret_id
          version = "latest"
        }
      }
    }
  }
}
```

GKE — External Secrets Operator 또는 Secret Manager CSI Driver.

---

## 5. 비용

```
$0.06 / secret·월 (active version)
$0.03 / 10K access
KMS 별도 (custom key 시)
Free: 10K access / month
```

→ AWS Secrets Manager 보다 약간 쌈.

---

## 6. Rotation

AWS 의 native rotation 같은 자동 X — Cloud Functions / Scheduler 로 자체 구현.
또는 Cloud SQL native — `cloudsql.iam_authentication` 통해 IAM auth 가 더 좋음.

---

## 7. 함정

- env var 로 노출 — process list / log
- delete = 7-30일 recovery window
- replication policy 변경 X
- KMS key 분실 = secret 영구 손실

---

## 8. 관련

- [[security]]
- [[iam]]
- [[cloud-kms]]
- [[../database/cloud-sql]]
