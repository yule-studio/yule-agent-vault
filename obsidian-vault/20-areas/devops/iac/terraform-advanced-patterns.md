---
title: "Terraform advanced patterns — composition / factory"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T09:50:00+09:00
tags: [devops, iac, terraform, advanced]
---

# Terraform advanced patterns — composition / factory

**[[iac|↑ iac]]**

---

## 1. composition (★ module 조합)

```hcl
# 작은 module 을 조합 → 큰 stack
module "vpc" {
  source = "./modules/vpc"
  cidr   = "10.0.0.0/16"
}

module "eks" {
  source     = "./modules/eks"
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnet_ids
}

module "rds" {
  source       = "./modules/rds"
  vpc_id       = module.vpc.vpc_id
  subnet_group = module.vpc.db_subnet_group
}

module "monitoring" {
  source       = "./modules/monitoring"
  cluster_name = module.eks.cluster_name
}
```

→ 각 module 은 단일 책임. dependency = output 참조.

---

## 2. factory pattern

```hcl
# environments/ 별 같은 config
module "factory" {
  source = "./modules/microservice"
  for_each = {
    user-service    = {replicas = 3,  cpu = "100m"}
    order-service   = {replicas = 5,  cpu = "200m"}
    payment-service = {replicas = 10, cpu = "500m"}
  }

  service_name = each.key
  replicas     = each.value.replicas
  cpu_request  = each.value.cpu
}
```

→ 한 module 로 N 개 비슷한 자원. 추가 = map 의 entry 추가.

---

## 3. module versioning

```hcl
module "vpc" {
  source  = "git::https://github.com/myorg/terraform-modules.git//vpc?ref=v1.2.3"
  # 또는 registry
  source  = "myorg/vpc/aws"
  version = "~> 1.2"

  # 또는 monorepo path
  source = "../../modules/vpc"   # relative
}
```

→ tag (`v1.2.3`) 또는 hash 사용. branch (`main`) 위험.

semver:
- MAJOR.MINOR.PATCH
- breaking 시 MAJOR ↑
- 새 feature 시 MINOR
- bugfix 시 PATCH

---

## 4. provider abstraction

```hcl
# 같은 module 이 여러 region 에 deploy
module "vpc_seoul" {
  source = "./modules/vpc"
  providers = {
    aws = aws.seoul
  }
}

module "vpc_tokyo" {
  source = "./modules/vpc"
  providers = {
    aws = aws.tokyo
  }
}

# providers
provider "aws" {
  alias  = "seoul"
  region = "ap-northeast-2"
}

provider "aws" {
  alias  = "tokyo"
  region = "ap-northeast-1"
}
```

→ multi-region 단순 배포.

---

## 5. self-reference (lifecycle)

```hcl
# ENI / IP 의 self-reference
resource "aws_eip" "web" {
  domain = "vpc"

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_instance" "web" {
  # ...
  lifecycle {
    create_before_destroy = true
    ignore_changes        = [ami]    # 옛 AMI 유지
  }
}

resource "aws_eip_association" "web" {
  instance_id   = aws_instance.web.id
  allocation_id = aws_eip.web.id
}
```

→ EIP 가 EC2 보다 먼저 → instance 교체 시 EIP 유지.

---

## 6. null_resource + provisioner

```hcl
# 외부 명령 실행 (terraform 외)
resource "null_resource" "wait_for_eks" {
  depends_on = [aws_eks_cluster.main]

  provisioner "local-exec" {
    command = "aws eks wait cluster-active --name ${aws_eks_cluster.main.name}"
  }

  triggers = {
    cluster_version = aws_eks_cluster.main.version
  }
}
```

→ 자원 type 이 없을 때 / 외부 command. **반드시 마지막 수단**.

---

## 7. conditional module

```hcl
# count 으로 module 비활성
module "monitoring" {
  count  = var.enable_monitoring ? 1 : 0
  source = "./modules/monitoring"
}

# for_each = {} 로 비활성
module "rds" {
  for_each = var.enable_rds ? {default = "yes"} : {}
  source   = "./modules/rds"
}
```

---

## 8. expand 패턴 (1 → N)

```hcl
# 1 stack 에서 N 환경
locals {
  envs = {
    dev = {
      vpc_cidr      = "10.10.0.0/16"
      instance_type = "t3.micro"
    }
    staging = {
      vpc_cidr      = "10.20.0.0/16"
      instance_type = "t3.small"
    }
    prod = {
      vpc_cidr      = "10.30.0.0/16"
      instance_type = "t3.large"
    }
  }
}

module "env" {
  for_each      = local.envs
  source        = "./modules/env"
  env_name      = each.key
  vpc_cidr      = each.value.vpc_cidr
  instance_type = each.value.instance_type
}
```

→ 한 apply 가 N 환경. 위험 (실수 시 전부 영향). Terragrunt 권장.

---

## 9. composition vs monolith

```
monolith:
  main.tf 안에 모든 resource → 작을 때 OK, 커지면 plan 느림

composition:
  여러 stack (vpc / eks / rds / monitoring 등) 분리
  각 stack 별도 state
  output → 다른 stack 의 data source

장점:
  - blast radius ↓
  - plan / apply 빠름
  - 팀별 책임
  - 의존성 명확
단점:
  - 변경 시 여러 stack apply
```

→ 시니어 = composition. 작은 = monolith.

---

## 10. output → input 패턴

```hcl
# Stack A (vpc)
output "vpc_id" {
  value = aws_vpc.main.id
}

# Stack B (eks)
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = {
    bucket = "my-tf-state"
    key    = "vpc/terraform.tfstate"
    region = "ap-northeast-2"
  }
}

module "eks" {
  vpc_id = data.terraform_remote_state.vpc.outputs.vpc_id
}
```

또는 explicit input (Terragrunt dependency):
```hcl
# terragrunt.hcl
dependency "vpc" {
  config_path = "../vpc"
}

inputs = {
  vpc_id = dependency.vpc.outputs.vpc_id
}
```

→ remote state 보다 dependency block 명확.

---

## 11. helper module (composite)

```hcl
# modules/web-stack/main.tf
# 작은 module 들의 조합 — 자주 쓰는 stack 추상화
module "alb" { source = "../alb"; ... }
module "asg" { source = "../asg"; ... }
module "rds" { source = "../rds"; ... }
output "endpoint" { value = module.alb.dns_name }
```

→ "복잡한 stack" 을 한 module 로. 단 너무 깊으면 디버그 어려움.

---

## 12. dry-run / sandbox

```bash
# 1. 새 환경 (test-userA) 자동 생성
TF_VAR_env="test-${USER}-$(date +%s)" terraform apply

# 2. 검증
./test/smoke.sh

# 3. 정리
terraform destroy -auto-approve
```

→ 시니어가 매번 sandbox 에서 검증 후 prod.

---

## 13. plan output diff (CI 의 PR 코멘트)

```yaml
# GitHub Actions
- run: terraform plan -no-color -out=plan.tfplan
- run: terraform show -no-color plan.tfplan > plan.txt
- uses: actions/github-script@v6
  with:
    script: |
      const fs = require('fs')
      const plan = fs.readFileSync('plan.txt', 'utf8')
      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: `## Terraform Plan\n\`\`\`\n${plan}\n\`\`\``
      })
```

→ PR 검토 시 변경 자동 visible.

---

## 14. 함정

1. **deeply nested module** — 디버깅 지옥.
2. **circular dependency between stacks** — split / refactor.
3. **provider alias 한 module 안에서 다중** — 헷갈림.
4. **null_resource 남용** — fragile (state 변경 추적 X).
5. **for_each 안의 for_each** — 가독성 X.
6. **factory module 의 한 entry 만 destroy** — 신중 (다른 영향 검토).
7. **monolith → 너무 큰 plan** — 분리 적기 놓침.

---

## 15. 관련

- [[iac|↑ iac]]
- [[terraform-modules]]
- [[terraform-functions-expressions]]
- [[terraform-workspaces]]
