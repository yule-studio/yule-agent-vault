---
title: "Terraform Cloud / Enterprise — managed Terraform"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T09:48:00+09:00
tags: [devops, iac, terraform, terraform-cloud]
---

# Terraform Cloud / Enterprise — managed Terraform

**[[iac|↑ iac]]**

---

## 1. 무엇

```
Terraform CLI 의 SaaS / Enterprise 호스팅 버전:
  - state 관리 (S3 안 필요)
  - remote apply (CI 에서 plan/apply)
  - VCS 통합 (GitHub → Cloud → apply)
  - team / org 관리
  - Sentinel (정책)
  - cost estimation
  - audit log
  - SSO
```

→ "Heroku for Terraform". 운영 부담 ↓.

---

## 2. 종류 / 가격

| | 무엇 | 비용 |
| --- | --- | --- |
| **Terraform Cloud Free** | 5 user, 기본 | $0 |
| **Terraform Cloud Standard** | team / SSO 등 | resource per hour |
| **Terraform Cloud Plus** | Sentinel / advanced | resource + user |
| **Terraform Enterprise** | self-host on-prem | 연 계약 |

---

## 3. 대안 (★ 시니어 결정)

| | 무엇 | 비교 |
| --- | --- | --- |
| **Terraform Cloud** | HashiCorp managed | 표준 |
| **Spacelift** | SaaS, multi-IaC (Pulumi/CDK) | 다양 도구 |
| **env0** | SaaS, 비슷 | Spacelift 경쟁 |
| **Atlantis** | OSS, self-host | 가격 0 |
| **GitHub Actions / GitLab CI** | 자체 pipeline | 단순, state 관리 별도 |
| **Scalr** | SaaS | 비슷 |

→ **소규모 / OSS = Atlantis**, **enterprise = Terraform Cloud / Spacelift**.

---

## 4. Terraform Cloud 시작

```bash
# CLI login
terraform login

# init 시 backend 설정
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "production"
    }
  }
}

terraform init
```

→ state 자동 cloud 저장. CI 도 OIDC token 으로 인증.

---

## 5. workspace (Cloud 의)

Cloud 의 workspace ≠ CLI workspace. 별개.

```
Cloud workspace = state + variables + VCS + run history 단위

예:
  myorg/prod-vpc
  myorg/prod-eks
  myorg/staging-vpc
```

각 workspace 별:
- VCS 연결 (GitHub repo + branch)
- variable set (env, secret)
- workflow (CLI / API / VCS-triggered)
- locking / approve

---

## 6. workflow 종류

### A. CLI-driven (basic)

```bash
terraform plan      # local plan + cloud state
terraform apply     # cloud 에서 실제 실행
```

### B. VCS-driven (★ 권장)

```
1. GitHub PR 생성
2. Cloud 가 자동 plan
3. PR comment 에 plan 결과
4. merge 시 자동 apply (또는 manual approve)
```

### C. API-driven

```bash
# 외부 system (CI) 에서 trigger
curl -X POST https://app.terraform.io/api/v2/runs \
    -H "Authorization: Bearer $TOKEN" \
    -d '...'
```

---

## 7. Sentinel (★ policy as code)

```python
# policies/no_public_s3.sentinel
import "tfplan/v2" as tfplan

s3_buckets = filter tfplan.resource_changes as _, rc {
    rc.type is "aws_s3_bucket" and
    (rc.change.actions contains "create" or rc.change.actions contains "update")
}

main = rule {
    all s3_buckets as _, rc {
        rc.change.after.acl is not "public-read" and
        rc.change.after.acl is not "public-read-write"
    }
}
```

→ apply 직전 정책 평가. 위반 시 block.

---

## 8. cost estimation

```
plan 시 자동:
  - 새 자원의 월 예상 비용
  - 변경 전후 차이
  - PR 의 cost 영향 보임

예:
  + aws_db_instance.main      ~$28/mo
  - aws_db_instance.old       -$28/mo
  Total: $0/mo change
```

→ PR 검토 시 비용 영향 visible.

---

## 9. team / RBAC

```
Organization 의 team:
  - Owners
  - Admin
  - Member

Workspace 의 permission:
  - read
  - plan
  - write
  - admin

→ 가장 시니어 만 prod admin
→ junior 는 dev plan / write
→ ops 는 모든 workspace admin
```

---

## 10. VCS-driven 예 (GitHub)

```yaml
# workspace 설정
VCS provider: GitHub
Repository: myorg/infra
Working directory: environments/prod
Branch: main
Trigger: file changes in directory

# 흐름
1. dev/foo branch → PR
2. Cloud 가 plan (sandbox account)
3. PR comment 에 plan
4. review + merge
5. main branch 가 변경 감지
6. plan + Sentinel
7. Sentinel pass → manual approve (또는 auto)
8. apply 실행
9. PR 에 결과 + slack 알림
```

---

## 11. Spacelift (대안 ★)

```
강점:
  - multi-IaC (Terraform / OpenTofu / Pulumi / CDK / Ansible / Kubernetes)
  - stack dependency (A → B → C)
  - drift detection
  - custom hooks (pre/post)
  - VCS / API integration

가격: workspace per hour (Cloud 보다 저렴)
```

---

## 12. Atlantis (★ OSS)

```yaml
# Atlantis 자체 host
docker run -p 4141:4141 \
    -e ATLANTIS_GH_USER=... \
    -e ATLANTIS_GH_TOKEN=... \
    -e ATLANTIS_REPO_ALLOWLIST="github.com/myorg/*" \
    runatlantis/atlantis
```

```
GitHub PR 의 comment:
  atlantis plan     → Atlantis 가 plan + 결과 comment
  atlantis apply    → apply (approve 후만)

장점:
  - 자체 host (비용 0)
  - 우리 CI 와 통합
  - state 는 별도 S3

단점:
  - Sentinel 같은 정책 X (별도 도구)
  - UI 약함
```

---

## 13. workspace 의 variable

```
1. Terraform variable (var.x)
   - sensitive flag → mask
   - HCL or string

2. Environment variable
   - AWS_ACCESS_KEY_ID 등
   - provider credential

3. Variable Set
   - 여러 workspace 공유 (common: AWS region)

4. dynamic credential (★)
   - OIDC 로 AWS Role assume
   - 영구 credential 없음
```

---

## 14. 함정

1. **CLI workspace 와 Cloud workspace 혼동** — 별개.
2. **무료 tier 의 resource limit** — 큰 workspace 비용 폭증.
3. **VCS-driven 의 default auto-apply** — review 없이 prod 변경.
4. **secret in VCS commit** — Cloud 가 read 하지만 git 평문.
5. **Sentinel 너무 엄격** — 모든 PR fail → 우회 시도.
6. **state migration 후 backup 없음** — 잃으면 끝.
7. **Atlantis self-host SPOF** — HA 또는 backup.

---

## 15. 관련

- [[iac|↑ iac]]
- [[terraform-basics]]
- [[terraform-security]]
- [[../cicd/pipeline-patterns|↗ CI/CD]]
