---
title: "실습 01 — Terraform AWS VPC + EC2"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:24:00+09:00
tags: [devops, iac, practice, terraform, aws]
---

# 실습 01 — Terraform AWS VPC + EC2

**[[practice|↑ practice]]**

> 1시간 — VPC + Public subnet + EC2 instance + SG.

---

## 1. 디렉토리

```
my-tf/
├── main.tf
├── variables.tf
├── outputs.tf
└── terraform.tfvars
```

---

## 2. main.tf

```hcl
terraform {
  required_version = ">= 1.6"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}

provider "aws" {
  region = var.region
}

data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = { Name = "${var.env}-vpc" }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "${var.env}-igw" }
}

resource "aws_subnet" "public" {
  count                   = 2
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.${count.index}.0/24"
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  tags = { Name = "${var.env}-public-${count.index}" }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
}

resource "aws_route_table_association" "public" {
  count          = 2
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_security_group" "web" {
  name        = "${var.env}-web-sg"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port = 22
    to_port   = 22
    protocol  = "tcp"
    cidr_blocks = ["YOUR_IP/32"]    # 본인 IP
  }
  ingress {
    from_port = 80
    to_port   = 80
    protocol  = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
    from_port = 0
    to_port   = 0
    protocol  = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

resource "aws_instance" "web" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.public[0].id
  vpc_security_group_ids = [aws_security_group.web.id]
  user_data = <<-EOF
    #!/bin/bash
    yum install -y nginx
    systemctl enable --now nginx
    echo "<h1>Hello Terraform!</h1>" > /usr/share/nginx/html/index.html
  EOF
  tags = { Name = "${var.env}-web" }
}
```

---

## 3. variables.tf / outputs.tf

```hcl
# variables.tf
variable "env" { type = string, default = "demo" }
variable "region" { type = string, default = "ap-northeast-2" }
```

```hcl
# outputs.tf
output "web_public_ip" {
  value = aws_instance.web.public_ip
}
```

---

## 4. 실행

```bash
terraform init
terraform plan
terraform apply
# 브라우저 → http://<public-ip>

terraform destroy   # 비용 방지
```

---

## 5. 다음

- 모듈 화 → [[../terraform-modules]]
- remote state → [[../terraform-state]]
- k8s (EKS) → 자체 module 또는 `terraform-aws-modules/eks/aws`
