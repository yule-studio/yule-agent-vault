---
title: "GCP IAM"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:30:00+09:00
tags:
  - gcp
  - security
  - iam
---

# GCP IAM

**[[security|↑ Security]]** · **[[../cloud-gcp|↑↑ GCP]]**

---

## 1. 핵심 구조

```
Organization → Folder → Project → Resource
                         + Member (user / SA / group) granted Role
```

| 개념 | 의미 |
| --- | --- |
| **Member** | user (gmail), Service Account, group, allUsers |
| **Role** | permission 묶음 (Basic / Predefined / Custom) |
| **Binding** | (member, role) on resource |
| **Service Account** | application identity |
| **Workload Identity** | K8s / GKE pod → SA |
| **Workload Identity Federation** | 외부 IdP (GitHub OIDC, AWS) → GCP |

---

## 2. Role 종류

| 종류 | 예 | 권장 |
| --- | --- | --- |
| **Basic** | `roles/owner`, `roles/editor`, `roles/viewer` | 광범위 — production X |
| **Predefined** | `roles/storage.objectViewer` | 대부분 |
| **Custom** | 자체 정의 | 정밀 제어 |

`gcloud iam roles list --filter='stage:GA'` 로 검색.

---

## 3. Service Account

```bash
gcloud iam service-accounts create myapp \
  --display-name="My App"

gcloud projects add-iam-policy-binding PROJECT \
  --member="serviceAccount:myapp@PROJECT.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"
```

```hcl
resource "google_service_account" "app" {
  account_id   = "myapp"
  display_name = "My App"
}

resource "google_project_iam_member" "app_storage" {
  project = var.project
  role    = "roles/storage.objectViewer"
  member  = "serviceAccount:${google_service_account.app.email}"
}
```

→ **key 만들기 X** — 가능한 Workload Identity / SA impersonation.

---

## 4. Workload Identity (GKE)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp
  annotations:
    iam.gke.io/gcp-service-account: myapp@PROJECT.iam.gserviceaccount.com
```

```bash
gcloud iam service-accounts add-iam-policy-binding myapp@PROJECT.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:PROJECT.svc.id.goog[NAMESPACE/myapp]"
```

→ pod 안 SDK 가 자동 SA 사용. key X.

---

## 5. Workload Identity Federation (외부)

GitHub Actions → GCP:

```hcl
resource "google_iam_workload_identity_pool" "github" {
  workload_identity_pool_id = "github"
}

resource "google_iam_workload_identity_pool_provider" "github" {
  workload_identity_pool_id          = google_iam_workload_identity_pool.github.id
  workload_identity_pool_provider_id = "github-oidc"
  oidc { issuer_uri = "https://token.actions.githubusercontent.com" }
  attribute_mapping = {
    "google.subject"       = "assertion.sub"
    "attribute.repository" = "assertion.repository"
  }
}

resource "google_service_account_iam_binding" "deploy" {
  service_account_id = google_service_account.deploy.name
  role               = "roles/iam.workloadIdentityUser"
  members            = ["principalSet://.../attribute.repository/myorg/myrepo"]
}
```

→ GitHub key 없이 OIDC 인증.

---

## 6. Audit Logs

기본 활성:
- **Admin Activity** — 항상
- **Data Access** — 활성 필요 (비용)
- **System Event** — 항상
- **Policy Denied** — 활성 필요

```bash
gcloud logging read 'logName:"cloudaudit.googleapis.com" AND resource.type="gce_instance"' --limit=10
```

자세히 → [[../observability/observability]]

---

## 7. 보안 원칙

1. **Least Privilege** — 광범위 Basic role X
2. **Service Account 별 권한 최소화** — 모든 권한을 한 SA 에 X
3. **Workload Identity** — key 발급 X
4. **MFA / 2FA** — admin 계정
5. **Organization Policy** — 회사 전체 강제
6. **VPC Service Controls** — data exfiltration 방지
7. **Audit Logs** + alert
8. **Recommender** — 미사용 권한 추천

---

## 8. SA Impersonation

```bash
gcloud auth login                          # 자기 계정
gcloud auth print-access-token --impersonate-service-account=...
```

→ 자기 계정으로 직접 사용 X, 임시 SA token 발급.

---

## 9. 함정

- `roles/owner` 남용 — production X
- SA key 발급 + git commit — 매우 흔한 leak
- service account key rotation 어려움 → Workload Identity
- Organization Policy 후 lockout (root access 보존)
- VPC SC 의 복잡성
- audit log 비용 — Data Access 활성 시 폭증

---

## 10. 관련

- [[security]]
- [[cloud-kms]]
- [[secret-manager]]
- [[../observability/observability]]
