---
title: "Terragrunt — DRY Terraform wrapper"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T09:52:00+09:00
tags: [devops, iac, terraform, terragrunt]
---

# Terragrunt — DRY Terraform wrapper

**[[iac|↑ iac]]**

---

## 1. 왜

```
Terraform 의 한계:
  - backend config 의 hardcode 반복
  - 여러 환경 / 다중 region → 복붙
  - 다른 account 의 module 호출 어려움
  - stack 간 dependency 명시 부족

→ Terragrunt 가 wrapper 로 해결:
  - DRY (Don't Repeat Yourself)
  - hierarchical config
  - dependency 명시
  - 동시 multi-stack apply
```

---

## 2. 설치

```bash
brew install terragrunt
# 또는
curl -L https://github.com/gruntwork-io/terragrunt/releases/download/v0.55.0/terragrunt_darwin_arm64 -o /usr/local/bin/terragrunt
chmod +x /usr/local/bin/terragrunt
terragrunt --version
```

---

## 3. 표준 구조 (★)

```
infra-live/
├── terragrunt.hcl                       ← root config (모든 env 공통)
├── _envcommon/                          ← module 공통 config
│   ├── vpc.hcl
│   ├── eks.hcl
│   └── rds.hcl
├── dev/
│   ├── env.hcl                          ← dev 공통
│   ├── ap-northeast-2/
│   │   ├── region.hcl                   ← region 공통
│   │   ├── vpc/
│   │   │   └── terragrunt.hcl           ← 실제 module 호출
│   │   ├── eks/
│   │   │   └── terragrunt.hcl
│   │   └── rds/
│   │       └── terragrunt.hcl
│   └── us-east-1/
│       └── ...
├── staging/
│   └── ...
└── prod/
    └── ...

infra-modules/                            ← 별도 repo (또는 monorepo)
├── vpc/
├── eks/
└── rds/
```

→ 환경 / region / module 의 3D 계층.

---

## 4. root terragrunt.hcl

```hcl
# infra-live/terragrunt.hcl
locals {
  account_name = "myorg"
  aws_region   = "ap-northeast-2"
}

remote_state {
  backend = "s3"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
  config = {
    bucket         = "myorg-tf-state"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "ap-northeast-2"
    encrypt        = true
    dynamodb_table = "myorg-tf-locks"
  }
}

generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
provider "aws" {
  region = "${local.aws_region}"

  default_tags {
    tags = {
      ManagedBy = "terragrunt"
      Account   = "${local.account_name}"
    }
  }
}
EOF
}
```

→ 모든 child 가 자동 backend + provider config 상속.

---

## 5. env 별 config

```hcl
# infra-live/prod/env.hcl
locals {
  env              = "prod"
  aws_account_id   = "111122223333"
  alert_email      = "ops@example.com"
}
```

```hcl
# infra-live/prod/ap-northeast-2/region.hcl
locals {
  region = "ap-northeast-2"
  azs    = ["ap-northeast-2a", "ap-northeast-2b", "ap-northeast-2c"]
}
```

---

## 6. module 호출 (★)

```hcl
# infra-live/prod/ap-northeast-2/vpc/terragrunt.hcl
include "root" {
  path = find_in_parent_folders()
}

include "envcommon" {
  path = "${dirname(find_in_parent_folders())}/_envcommon/vpc.hcl"
}

locals {
  env_vars    = read_terragrunt_config(find_in_parent_folders("env.hcl"))
  region_vars = read_terragrunt_config(find_in_parent_folders("region.hcl"))
}

terraform {
  source = "git::https://github.com/myorg/infra-modules.git//vpc?ref=v1.2.3"
}

inputs = {
  env  = local.env_vars.locals.env
  cidr = "10.30.0.0/16"
  azs  = local.region_vars.locals.azs
}
```

```hcl
# _envcommon/vpc.hcl
inputs = {
  enable_nat_gateway   = true
  enable_vpn_gateway   = false
  enable_dns_hostnames = true
}
```

→ root → envcommon → env.hcl → region.hcl → 실제 → merge.

---

## 7. dependency block (★)

```hcl
# infra-live/prod/ap-northeast-2/eks/terragrunt.hcl
include "root" {
  path = find_in_parent_folders()
}

terraform {
  source = "git::...//eks?ref=v1.0.0"
}

dependency "vpc" {
  config_path = "../vpc"

  # vpc 가 plan 실패 시 의 fallback
  mock_outputs = {
    vpc_id     = "mock-vpc-id"
    subnet_ids = ["mock-subnet-1", "mock-subnet-2"]
  }
}

inputs = {
  vpc_id     = dependency.vpc.outputs.vpc_id
  subnet_ids = dependency.vpc.outputs.subnet_ids
}
```

→ `terragrunt apply` 시 vpc 가 먼저 → eks. dependency 자동 ordering.

---

## 8. run-all (★ 동시)

```bash
# 전체 stack 한번에
terragrunt run-all plan
terragrunt run-all apply

# 특정 dir 만
cd prod/ap-northeast-2
terragrunt run-all apply

# 의존성 순서대로
# vpc → eks → monitoring 순
```

→ dependency 따라 자동 ordering + 가능한 만큼 parallel.

---

## 9. hook (★)

```hcl
terraform {
  source = "..."

  before_hook "format" {
    commands = ["plan", "apply"]
    execute  = ["terraform", "fmt", "-recursive"]
  }

  before_hook "tfsec" {
    commands = ["plan"]
    execute  = ["tfsec", "."]
  }

  after_hook "notify" {
    commands = ["apply"]
    execute  = ["curl", "-X", "POST", "https://hooks.slack.com/...", "-d", "deployed!"]
    run_on_error = false
  }

  error_hook "rollback" {
    commands = ["apply"]
    execute  = ["./scripts/rollback.sh"]
    on_errors = [".*"]
  }
}
```

---

## 10. 환경 변수 / secret

```hcl
locals {
  # AWS Secrets Manager 에서 read
  db_password = run_cmd("aws", "secretsmanager", "get-secret-value", "--secret-id", "prod/db/password", "--query", "SecretString", "--output", "text")
}

inputs = {
  db_password = local.db_password
}
```

→ Vault / SOPS 도 비슷하게.

---

## 11. 단일 backend → 여러 state

```
한 S3 bucket (myorg-tf-state) 에:
  dev/ap-northeast-2/vpc/terraform.tfstate
  dev/ap-northeast-2/eks/terraform.tfstate
  prod/ap-northeast-2/vpc/terraform.tfstate
  prod/us-east-1/vpc/terraform.tfstate
  ...

→ key 가 path_relative_to_include() 로 자동
```

→ state 분리 단순.

---

## 12. 실전 흐름

```
1. dev 에 새 자원 추가
   - infra-live/dev/ap-northeast-2/new-stack/terragrunt.hcl 작성
   - cd 해서 terragrunt apply

2. staging 도 (config 만 다름)
   - infra-live/staging/.../new-stack/terragrunt.hcl
   - inputs 조정 (size 등)

3. prod
   - 같은 작업
   - PR review

4. multi-region
   - infra-live/prod/us-east-1/new-stack/terragrunt.hcl 복사
   - region.hcl 만 다름
```

---

## 13. CI 통합

```yaml
# GitHub Actions
- name: terragrunt plan
  run: |
    cd prod/ap-northeast-2
    terragrunt run-all plan --terragrunt-non-interactive

- name: terragrunt apply
  if: github.ref == 'refs/heads/main'
  run: |
    cd prod/ap-northeast-2
    terragrunt run-all apply --terragrunt-non-interactive --auto-approve
```

→ 사람 review 후 main merge → auto-apply.

---

## 14. 함정

1. **terragrunt 와 terraform workspaces 같이** — 충돌. 하나만.
2. **mock_outputs 안 함** — plan 실패 cascade.
3. **dependency 순환** — graph 분리.
4. **run-all 의 parallel 위험** — 동시 API rate limit / 순서 의존.
5. **`terragrunt destroy` 잘못 위치** — 전체 stack destroy.
6. **너무 깊은 include** — 디버그 어려움.
7. **hook 의 secret echo** — log 노출.

---

## 15. 관련

- [[iac|↑ iac]]
- [[terraform-workspaces]]
- [[terraform-advanced-patterns]]
- [[terraform-modules]]
