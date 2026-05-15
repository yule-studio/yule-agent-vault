---
title: "Terraform state — remote backend + locking"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:08:00+09:00
tags: [devops, iac, terraform, state]
---

# Terraform state — remote backend + locking

**[[iac|↑ iac]]**

---

## 1. state 가 뭔가

- terraform 의 "현재 상태" (어떤 자원이 있는지 추적).
- local default: `terraform.tfstate` (git ignore).
- team / CI 면 **remote backend** 필수.

---

## 2. S3 + DynamoDB lock (AWS)

```hcl
terraform {
  backend "s3" {
    bucket         = "tfstate-myorg"
    key            = "prod/main.tfstate"
    region         = "ap-northeast-2"
    encrypt        = true
    dynamodb_table = "tfstate-lock"
  }
}
```

```bash
# 사전 — S3 bucket + DynamoDB table 생성 (bootstrap)
aws s3 mb s3://tfstate-myorg --region ap-northeast-2
aws dynamodb create-table \
    --table-name tfstate-lock \
    --attribute-definitions AttributeName=LockID,AttributeType=S \
    --key-schema AttributeName=LockID,KeyType=HASH \
    --billing-mode PAY_PER_REQUEST
```

---

## 3. GCS (GCP) / Azure Blob

```hcl
backend "gcs" {
  bucket = "tfstate-myorg"
  prefix = "prod"
}

# 또는
backend "azurerm" {
  resource_group_name  = "rg-tfstate"
  storage_account_name = "tfstatemyorg"
  container_name       = "tfstate"
  key                  = "prod.tfstate"
}
```

---

## 4. Terraform Cloud / Spacelift

```hcl
terraform {
  cloud {
    organization = "myorg"
    workspaces { name = "prod" }
  }
}
```

→ state + run + 협업 통합.

---

## 5. workspace (환경 분리)

```bash
terraform workspace new dev
terraform workspace new prod
terraform workspace select prod
terraform workspace list
```

```hcl
resource "aws_s3_bucket" "logs" {
  bucket = "logs-${terraform.workspace}"
}
```

→ 본 vault 는 **별도 디렉토리** (envs/prod/, envs/dev/) 권장 (workspace 보다 명확).

---

## 6. state 안 secret

- secret 이 state 에 평문 — `apply` 결과 노출 가능.
- backend encryption (S3 SSE / GCS / Azure) + restricted IAM.
- TF_LOG=DEBUG 시 secret 도 log → CI 에서 조심.

---

## 7. import (외부 자원 등록)

```bash
terraform import aws_instance.web i-abc123def
```

또는 `terraform-aws-import` 도구 / `terraformer`.

---

## 8. 함정

1. **local state commit** → 다중 user 충돌 + secret leak.
2. **lock table 없음** → 동시 apply → state corruption.
3. **state 직접 수정** → drift.
4. **`terraform refresh` 자주** → state 변경 추적 어려움.
5. **bucket versioning off** → 손상 시 복구 불가.

---

## 9. 관련

- [[iac|↑ iac]]
- [[terraform-basics]]
- [[best-practices]]
