---
title: "Terraform modules — 재사용"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:06:00+09:00
tags: [devops, iac, terraform, module]
---

# Terraform modules — 재사용

**[[iac|↑ iac]]**

---

## 1. 모듈 구조

```
modules/
└── vpc/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    └── README.md
```

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "this" {
  cidr_block = var.cidr
  tags       = merge(var.tags, { Name = var.name })
}

resource "aws_subnet" "public" {
  count = length(var.azs)
  vpc_id            = aws_vpc.this.id
  cidr_block        = cidrsubnet(var.cidr, 8, count.index)
  availability_zone = var.azs[count.index]
  tags              = { Name = "${var.name}-public-${count.index}" }
}
```

```hcl
# modules/vpc/variables.tf
variable "name"   { type = string }
variable "cidr"   { type = string }
variable "azs"    { type = list(string) }
variable "tags"   { type = map(string), default = {} }
```

```hcl
# modules/vpc/outputs.tf
output "vpc_id"     { value = aws_vpc.this.id }
output "subnet_ids" { value = aws_subnet.public[*].id }
```

---

## 2. 사용

```hcl
# main.tf
module "prod_vpc" {
  source = "./modules/vpc"
  name   = "prod"
  cidr   = "10.0.0.0/16"
  azs    = ["ap-northeast-2a", "ap-northeast-2b"]
  tags   = { Env = "prod" }
}

resource "aws_instance" "web" {
  subnet_id = module.prod_vpc.subnet_ids[0]
}
```

---

## 3. 외부 module (registry)

```hcl
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  version = "5.5.0"

  name = "prod"
  cidr = "10.0.0.0/16"
  azs  = ["ap-northeast-2a", "ap-northeast-2b"]
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets = ["10.0.10.0/24", "10.0.20.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = true
}
```

→ `registry.terraform.io` 의 공식 / community module.

---

## 4. layout 권장 (환경 분리)

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
└── README.md
```

→ env 별 state 분리.

---

## 5. 함정

1. **module version 미고정** → 다른 버전 → 다른 결과.
2. **module 안에 provider 정의** → 외부 override 어려움.
3. **`null` / empty list** — 다른 동작.
4. **module 의 너무 깊은 nesting** → debug 어려움.
5. **module 가 단일 자원** — overhead.

---

## 6. 관련

- [[iac|↑ iac]]
- [[terraform-basics]]
- [[best-practices]]
