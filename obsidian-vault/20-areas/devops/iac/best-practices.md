---
title: "IaC best practices"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:18:00+09:00
tags: [devops, iac, best-practice]
---

# IaC best practices

**[[iac|↑ iac]]**

---

## 1. 디렉토리 layout (Terraform)

```
.
├── modules/
│   ├── vpc/
│   ├── eks/
│   └── rds/
├── envs/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   ├── staging/
│   └── prod/
└── scripts/
```

→ env 별 state 분리, module 재사용.

---

## 2. naming convention

```hcl
# resource type-prefix 생략 (이미 type 명시)
resource "aws_instance" "web" {}           # NOT aws_instance.aws_web

# kebab / snake 일관
locals {
  name_prefix = "${var.env}-${var.app}"
}
```

---

## 3. tagging 표준

```hcl
locals {
  common_tags = {
    Env       = var.env
    Owner     = var.owner
    ManagedBy = "terraform"
    Repo      = "github.com/myorg/infra"
  }
}

resource "aws_instance" "web" {
  tags = merge(local.common_tags, { Name = "web" })
}
```

---

## 4. PR 워크플로 (Atlantis 또는 Terraform Cloud)

```
PR open → terraform plan 자동 → PR 코멘트
PR review → approve
PR merge → terraform apply 자동 (또는 manual)
```

---

## 5. scan / lint

| 도구 | 검출 |
| --- | --- |
| **tflint** | best practice |
| **tfsec** | 보안 |
| **Checkov** | compliance + sec |
| **Terrascan** | 정책 |
| **OPA / Conftest** | 자체 정책 (rego) |

```bash
tflint
tfsec .
checkov -d .
```

---

## 6. secret 관리

```hcl
# X — 평문
variable "db_password" {
  default = "supersecret"
}

# O — env / Vault
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/db-password"
}

resource "aws_db_instance" "main" {
  password = data.aws_secretsmanager_secret_version.db.secret_string
}
```

---

## 7. import vs new

```bash
# 외부에서 만든 자원 → terraform 으로 가져오기
terraform import aws_instance.web i-abc123
```

→ console 클릭 으로 만든 자원은 import + state 동기.

---

## 8. drift detection

```bash
# 매일 cron — drift 감지
terraform plan -detailed-exitcode
# exit 2 = drift
```

→ Atlantis / Terraform Cloud 가 자동.

---

## 9. 함정

1. **module 자체에 provider** → override 어려움.
2. **count 사용 + reorder** → destroy/recreate.
3. **state 직접 수정** → drift.
4. **secret tfvars commit** → leak.
5. **destroy prod** — prevent_destroy + workspace + IAM.

---

## 10. 관련

- [[iac|↑ iac]]
- [[terraform-basics]]
- [[terraform-modules]]
- [[terraform-state]]
- [[pitfalls]]
