---
title: "Terraform 기초 ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:04:00+09:00
tags: [devops, iac, terraform]
---

# Terraform 기초 ★

**[[iac|↑ iac]]**

---

## 1. 핵심 명령

```bash
terraform init             # provider download + backend
terraform plan             # 미리보기 (변경 없음)
terraform plan -out=p
terraform apply p          # plan 적용
terraform destroy          # 모두 삭제
terraform fmt              # 포맷
terraform validate         # 문법
terraform import aws_instance.web i-abc123    # 외부 자원 등록
terraform state list
terraform state show aws_instance.web
```

---

## 2. main.tf 예시 (AWS VPC + EC2)

```hcl
terraform {
  required_version = ">= 1.6"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
  backend "s3" {
    bucket = "tfstate-myorg"
    key    = "prod/main.tfstate"
    region = "ap-northeast-2"
    dynamodb_table = "tfstate-lock"
  }
}

provider "aws" {
  region = "ap-northeast-2"
}

variable "env" {
  type    = string
  default = "prod"
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags       = { Name = "${var.env}-vpc" }
}

resource "aws_subnet" "public" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
}

data "aws_availability_zones" "available" {}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.public[0].id

  tags = {
    Name = "${var.env}-web"
  }
}

output "instance_ip" {
  value = aws_instance.web.public_ip
}
```

---

## 3. 변수 (variables.tf + tfvars)

```hcl
# variables.tf
variable "env" {
  type    = string
  default = "dev"
}

variable "instance_count" {
  type    = number
  default = 2
}

variable "tags" {
  type    = map(string)
  default = { Owner = "team-a" }
}
```

```hcl
# terraform.tfvars  (git ignore — secret 포함 시)
env = "prod"
instance_count = 5
```

또는 env var: `TF_VAR_env=prod`.

---

## 4. count vs for_each

```hcl
# count — index 기반
resource "aws_instance" "web" {
  count = 3
  ami   = "ami-..."
  tags  = { Name = "web-${count.index}" }
}

# for_each — map / set 기반 (안정 — 순서 변경 영향 X)
resource "aws_instance" "web" {
  for_each = toset(["web1", "web2", "web3"])
  ami      = "ami-..."
  tags     = { Name = each.key }
}
```

→ for_each 권장 (count 는 reorder 시 destroy/recreate).

---

## 5. lifecycle

```hcl
resource "aws_db_instance" "db" {
  # ...
  lifecycle {
    prevent_destroy       = true             # 실수 destroy 방지
    ignore_changes        = [password]       # 외부 변경 무시
    create_before_destroy = true             # 교체 시 새 자원 먼저
  }
}
```

---

## 6. data source (read-only)

```hcl
data "aws_vpc" "default" {
  default = true
}

data "aws_caller_identity" "current" {}

resource "aws_s3_bucket" "logs" {
  bucket = "logs-${data.aws_caller_identity.current.account_id}"
}
```

---

## 7. 함정

1. **state secret commit** → AK/SK 노출 → remote backend 필수.
2. **state file 직접 수정** → drift.
3. **provider version 미고정** → 다른 결과.
4. **count 사용 + reorder** → destroy / recreate.
5. **`terraform destroy` 실수 prod** → prevent_destroy + workspace 분리.

---

## 8. 관련

- [[iac|↑ iac]]
- [[terraform-modules]]
- [[terraform-state]]
- [[best-practices]]
