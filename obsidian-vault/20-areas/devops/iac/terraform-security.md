---
title: "Terraform security — sensitive / encryption / secret"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T09:42:00+09:00
tags: [devops, iac, terraform, security]
---

# Terraform security — sensitive / encryption / secret

**[[iac|↑ iac]]**

---

## 1. 위험 영역

```
1. tfstate 의 secret 평문 (DB password, API key)
2. *.tfvars 가 git 커밋
3. provider credential 평문
4. plan 출력 의 secret 노출 (CI log)
5. drift / public 자원 생성
6. IAM `*` (overly permissive)
7. 의도하지 않은 destroy
8. supply chain (3rd-party module)
```

---

## 2. sensitive variable

```hcl
variable "db_password" {
  type      = string
  sensitive = true            # plan 출력에 "<sensitive>" 로 마스킹
}

output "connection_string" {
  value     = "postgres://admin:${var.db_password}@host/db"
  sensitive = true            # output 도 mask
}

# state 에는 여전히 평문 ★ — state encryption 필요
```

→ **sensitive 는 출력만 막음**. state 평문은 별도 보호.

---

## 3. tfstate encryption

```hcl
# S3 backend with KMS
terraform {
  backend "s3" {
    bucket         = "my-tf-state"
    key            = "prod/terraform.tfstate"
    region         = "ap-northeast-2"
    encrypt        = true                              # ★ S3-side encryption
    kms_key_id     = "arn:aws:kms:..."                 # KMS CMK
    dynamodb_table = "my-tf-locks"                     # 동시 실행 lock
  }
}
```

→ S3 bucket 의 IAM 정책으로 접근 제한 + KMS 로 데이터 암호화.

---

## 4. credential 관리 (★)

```bash
# ❌ 절대 X
provider "aws" {
  access_key = "AKIA..."          # 평문 X
  secret_key = "..."
}

# ✅ 환경 변수
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...

# ✅ AWS profile
provider "aws" {
  profile = "prod"
}

# ✅ AWS SSO
aws sso login --profile prod

# ✅ Role assume
provider "aws" {
  assume_role {
    role_arn = "arn:aws:iam::123:role/terraform"
  }
}

# ✅ CI = OIDC (★ 표준)
# GitHub Actions → AWS Role 임시 token, 영구 key 없음
```

---

## 5. tfvars / .gitignore

```bash
# .gitignore
*.tfstate
*.tfstate.*
*.tfvars
!example.tfvars
.terraform/
.terraform.lock.hcl
crash.log
override.tf
override.tf.json
*_override.tf

# 또한 git history 확인
git secrets --scan       # AWS / GCP key 검출
gitleaks detect
```

---

## 6. secret 어디서 가져오나

```hcl
# A. Vault
data "vault_generic_secret" "db" {
  path = "secret/prod/db"
}

resource "aws_db_instance" "main" {
  password = data.vault_generic_secret.db.data["password"]
}

# B. AWS Secrets Manager
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/db/password"
}

resource "aws_db_instance" "main" {
  password = jsondecode(data.aws_secretsmanager_secret_version.db.secret_string)["password"]
}

# C. SOPS
# 파일 자체를 SOPS 로 암호화 + git commit
data "sops_file" "config" {
  source_file = "secrets.enc.yaml"
}
```

→ 평문 tfvars X. 동적 secret store 에서 read.

---

## 7. random password

```hcl
resource "random_password" "db" {
  length           = 32
  special          = true
  override_special = "!#$%&*-_=+"
}

resource "aws_db_instance" "main" {
  password = random_password.db.result
}

resource "aws_secretsmanager_secret_version" "db" {
  secret_id     = aws_secretsmanager_secret.db.id
  secret_string = random_password.db.result
}
```

→ password 가 state 에 평문이지만 application 은 Secrets Manager 에서 read.

---

## 8. IAM least privilege

```hcl
# ❌ overly permissive
resource "aws_iam_policy" "bad" {
  policy = jsonencode({
    Statement = [{
      Effect   = "Allow"
      Action   = "*"
      Resource = "*"
    }]
  })
}

# ✅ specific
resource "aws_iam_policy" "good" {
  policy = jsonencode({
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject", "s3:PutObject"]
      Resource = "${aws_s3_bucket.uploads.arn}/uploads/*"
    }]
  })
}

# 또는 AWS managed policy
data "aws_iam_policy_document" "read_only" {
  source_policy_documents = [
    data.aws_iam_policy_document.s3_read.json,
    data.aws_iam_policy_document.kms_decrypt.json
  ]
}
```

---

## 9. encryption at rest 강제

```hcl
# S3
resource "aws_s3_bucket_server_side_encryption_configuration" "main" {
  bucket = aws_s3_bucket.main.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.main.id
    }
  }
}

resource "aws_s3_bucket_public_access_block" "main" {
  bucket                  = aws_s3_bucket.main.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# RDS
resource "aws_db_instance" "main" {
  storage_encrypted = true
  kms_key_id        = aws_kms_key.main.arn
}

# EBS
resource "aws_ebs_volume" "main" {
  encrypted = true
}
```

---

## 10. 보안 검증 (CI)

```yaml
# GitHub Actions
- run: tfsec .
- run: checkov -d .
- run: terraform-compliance -f features/ -p plan.json

# tfsec config
# .tfsec.yml
exclude:
  - aws-s3-enable-bucket-logging  # 명시적 예외만
```

---

## 11. supply chain (★)

```hcl
# ❌ 위험
module "vpc" {
  source = "github.com/random-user/terraform-vpc"     # 검증 X
}

# ✅ HashiCorp registry (검증 됨)
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.5.0"
}

# ✅ private registry (회사 module)
module "vpc" {
  source  = "app.terraform.io/myorg/vpc/aws"
  version = "1.0.0"
}

# ✅ git tag pinning
module "vpc" {
  source = "git::https://github.com/myorg/tf-vpc.git?ref=v1.2.3"
}
```

→ **branch 참조 X** (변경됨). tag / commit hash 만.

---

## 12. drift / unauthorized change 탐지

```bash
# 정기 drift 검출
terraform plan -detailed-exitcode
# exit code:
#   0 = no changes
#   1 = error
#   2 = drift 있음

# cron 으로 매시간 → drift 시 slack alert
```

```hcl
# 자체 protection
resource "aws_iam_user" "admin" {
  lifecycle {
    prevent_destroy = true   # destroy 시도 거부
  }
}
```

---

## 13. audit log

```hcl
# Terraform Cloud / Enterprise: 자동 audit
# OSS: state file lock / CloudTrail 로 추적

# 모든 apply 가:
# 1. PR 으로
# 2. 2-person review
# 3. CI 자동 apply (manual apply 금지)
# 4. CloudTrail 의 IAM Role 가 누가
```

---

## 14. 함정

1. **tfvars git 커밋** — secret 누출.
2. **state 평문 + S3 encryption off** — secret 노출.
3. **provider credential 평문 코드** — git history.
4. **CI log 의 plan output** — `-no-color`, mask 적용.
5. **branch 의 module source** — 변경 위험.
6. **IAM `*`** — over-permission.
7. **destroy 자동화** — prod prevent_destroy 또는 manual gate.
8. **drift 무시** — 보안 사고 신호 (누가 manual 변경).

---

## 15. 관련

- [[iac|↑ iac]]
- [[terraform-state]]
- [[../security-ops/secrets-management|↗ secrets-management]]
- [[../security-ops/supply-chain-security|↗ supply chain]]
