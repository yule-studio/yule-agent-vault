---
title: "Terraform workspaces — 환경 분리"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T09:35:00+09:00
tags: [devops, iac, terraform, workspaces]
---

# Terraform workspaces — 환경 분리

**[[iac|↑ iac]]**

---

## 1. 무엇

```
한 codebase 에서 여러 state 유지:
  workspace "dev"        → state file dev
  workspace "staging"    → state file staging
  workspace "prod"       → state file prod
```

→ 같은 코드, 다른 변수, 다른 인프라.

---

## 2. 명령

```bash
terraform workspace list
terraform workspace new staging
terraform workspace select staging
terraform workspace show
terraform workspace delete dev

# 또는
TF_WORKSPACE=prod terraform plan
```

---

## 3. 코드에서 참조

```hcl
locals {
  env = terraform.workspace      # "dev" / "staging" / "prod"

  config = {
    dev = {
      instance_type = "t3.micro"
      replicas      = 1
    }
    staging = {
      instance_type = "t3.small"
      replicas      = 2
    }
    prod = {
      instance_type = "t3.large"
      replicas      = 5
    }
  }
}

resource "aws_instance" "web" {
  count         = local.config[local.env].replicas
  instance_type = local.config[local.env].instance_type
  tags = {
    Environment = local.env
  }
}
```

→ `terraform.workspace` 가 magic var.

---

## 4. backend (state) 분리

```hcl
# S3 backend 예
terraform {
  backend "s3" {
    bucket = "my-tf-state"
    key    = "myapp/terraform.tfstate"
    region = "ap-northeast-2"
  }
}
```

workspace 사용 시 실제 키:
```
s3://my-tf-state/env:/dev/myapp/terraform.tfstate
s3://my-tf-state/env:/staging/myapp/terraform.tfstate
s3://my-tf-state/env:/prod/myapp/terraform.tfstate
```

→ 자동으로 `env:/<workspace>/` prefix.

---

## 5. workspaces 의 한계 (★ 시니어 결정)

```
✗ default workspace 외 backend config 다르게 불가
✗ workspace 별 다른 module / provider version 어려움
✗ workspace 별 다른 AWS account 불가능 (한 backend = 한 account assumption)
✗ workspace switch 가 실수로 prod 영향 위험

✓ 같은 코드, 같은 cloud account, 같은 region 의 환경 분리 OK
✓ dev / staging / qa 같이 비슷한 환경
```

→ **prod 와 dev 가 다른 AWS account 면 workspaces 부적합** → directory 또는 Terragrunt.

---

## 6. directory 패턴 (대안 ★ 권장)

```
infra/
├── modules/
│   ├── vpc/
│   ├── eks/
│   └── rds/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── backend.tf      ← dev S3 bucket
│   │   └── terraform.tfvars
│   ├── staging/
│   │   ├── main.tf
│   │   ├── backend.tf      ← staging S3 bucket
│   │   └── terraform.tfvars
│   └── prod/
│       ├── main.tf
│       ├── backend.tf      ← prod S3 bucket
│       └── terraform.tfvars
└── README.md
```

```hcl
# environments/prod/main.tf
module "vpc" {
  source = "../../modules/vpc"
  cidr   = "10.0.0.0/16"
}
```

→ 각 환경이 독립. 다른 account / region / backend 가능.

---

## 7. Terragrunt (★ 권장 + DRY)

```
infra/
├── terragrunt.hcl                     ← 공통
├── _envcommon/
│   └── vpc.hcl                        ← 공통 module
├── dev/
│   ├── terragrunt.hcl                 ← env config
│   ├── ap-northeast-2/
│   │   └── vpc/
│   │       └── terragrunt.hcl
├── prod/
│   ├── terragrunt.hcl
│   └── ap-northeast-2/
│       └── vpc/
│           └── terragrunt.hcl
```

```hcl
# infra/terragrunt.hcl (root)
remote_state {
  backend = "s3"
  config = {
    bucket = "my-tf-state-${get_env("ENV")}"
    key    = "${path_relative_to_include()}/terraform.tfstate"
    region = "ap-northeast-2"
  }
}
```

```hcl
# prod/ap-northeast-2/vpc/terragrunt.hcl
include "root" {
  path = find_in_parent_folders()
}

terraform {
  source = "../../../modules/vpc"
}

inputs = {
  cidr     = "10.0.0.0/16"
  env      = "prod"
  region   = "ap-northeast-2"
}
```

→ DRY (Don't Repeat Yourself). 거대한 multi-account / multi-region 표준.

---

## 8. 비교 (★ 시니어 결정 매트릭스)

| | workspaces | directory | Terragrunt |
| --- | --- | --- | --- |
| 학습 | 쉬움 | 중간 | 어려움 |
| 환경 격리 | 약함 | 강함 | 강함 |
| 다중 account | ✗ | ✓ | ✓ |
| DRY | 좋음 | 약함 (복붙) | ★ 가장 좋음 |
| state 격리 | auto | manual | auto |
| 도구 추가 | X (terraform 내장) | X | terragrunt 설치 |
| GitOps 친화 | OK | 좋음 | 좋음 |
| 사용 | dev/staging 같은 account | 작은 multi-env | 대규모 |

→ **권장:** 소규모 / dev-staging 같은 account = **workspaces**. 그 외 모두 = **Terragrunt**.

---

## 9. workspaces 안전 패턴

```hcl
# prod 변경 막기
locals {
  allowed_workspaces = ["dev", "staging"]
  is_safe = contains(local.allowed_workspaces, terraform.workspace)
}

resource "null_resource" "guard" {
  count = local.is_safe ? 0 : 1
  provisioner "local-exec" {
    command = "echo 'ERROR: workspace ${terraform.workspace} is restricted' && exit 1"
  }
}
```

또는 `terraform plan` 의 명시 `-var-file` 강제 — CI 에서 강제.

---

## 10. 실수 방지

```bash
# 1. workspace 항상 확인 후 명령
terraform workspace show

# 2. shell prompt 에 workspace 표시
export PS1='[\u@\h tf:$(terraform workspace show 2>/dev/null) \W]\$ '

# 3. CI 에서만 prod
# CI script:
TF_WORKSPACE=prod terraform plan -var-file=prod.tfvars

# 4. -auto-approve 금지 (prod)
```

---

## 11. 함정

1. **workspaces + 다른 account** — backend 한정, 위험.
2. **default workspace 사용** — 이름 명확 X. 항상 명시.
3. **prod 에 -auto-approve** — review 없이 destroy.
4. **workspace switch 후 plan 안 함** — 실수.
5. **CI / dev 의 workspace 혼동** — 같은 backend 의 state 변경 race.
6. **state 의 workspace 단순 prefix** — backup / migration 시 헷갈림.
7. **workspaces 가 module 인 줄** — 단순 state 분리.

---

## 12. 관련

- [[iac|↑ iac]]
- [[terraform-basics]]
- [[terraform-state]]
- [[terraform-advanced-patterns]]
