---
title: "Terraform functions / expressions — 자주 쓰는 패턴"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T09:37:00+09:00
tags: [devops, iac, terraform, functions]
---

# Terraform functions / expressions — 자주 쓰는 패턴

**[[iac|↑ iac]]**

---

## 1. 변수 / locals / output

```hcl
variable "env" {
  type        = string
  description = "환경 (dev/staging/prod)"
  default     = "dev"
  validation {
    condition     = contains(["dev", "staging", "prod"], var.env)
    error_message = "env must be dev/staging/prod"
  }
}

locals {
  prefix = "${var.env}-myapp"
  common_tags = {
    Environment = var.env
    ManagedBy   = "terraform"
  }
}

output "vpc_id" {
  value       = aws_vpc.main.id
  description = "VPC ID"
  sensitive   = false
}
```

---

## 2. for / for_each

```hcl
# list 의 모든 element 변환
output "subnet_ids" {
  value = [for s in aws_subnet.private : s.id]
}

# map 만들기
output "instance_ips" {
  value = {
    for i in aws_instance.web : i.tags.Name => i.private_ip
  }
}

# filter + transform
output "public_subnets" {
  value = [for s in aws_subnet.all : s.id if s.map_public_ip_on_launch]
}
```

```hcl
# count vs for_each
# count = 단순 N 개
resource "aws_instance" "old" {
  count         = 3
  instance_type = "t3.micro"
  tags          = {Name = "web-${count.index}"}
}

# for_each = 명시 set / map (★ 권장)
resource "aws_instance" "new" {
  for_each      = toset(["web", "api", "worker"])
  instance_type = "t3.micro"
  tags          = {Name = each.key}
}

# map 으로 다른 config
resource "aws_instance" "by_role" {
  for_each      = {
    web    = {type = "t3.micro", count = 2}
    api    = {type = "t3.small", count = 3}
    worker = {type = "t3.medium", count = 1}
  }
  instance_type = each.value.type
  tags          = {Role = each.key}
}
```

→ **for_each 권장** — index 가 아닌 key 라 manifest stable.

---

## 3. dynamic block

```hcl
# 여러 ingress rule 동적 생성
resource "aws_security_group" "web" {
  name = "web-sg"
  vpc_id = aws_vpc.main.id

  dynamic "ingress" {
    for_each = var.allowed_ports
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = ingress.value.cidrs
    }
  }
}

# var
variable "allowed_ports" {
  default = [
    {port = 80,  cidrs = ["0.0.0.0/0"]},
    {port = 443, cidrs = ["0.0.0.0/0"]},
    {port = 22,  cidrs = ["10.0.0.0/8"]}
  ]
}
```

---

## 4. conditional expression

```hcl
# ternary
resource "aws_instance" "web" {
  instance_type = var.env == "prod" ? "t3.large" : "t3.micro"
}

# count 으로 conditional 자원
resource "aws_db_instance" "replica" {
  count = var.env == "prod" ? 1 : 0
  # ...
}

# nullable + conditional
encryption_key = var.encrypt ? aws_kms_key.main.arn : null
```

---

## 5. 자주 쓰는 built-in function

```hcl
# string
upper("hello")                  # "HELLO"
lower("HELLO")                  # "hello"
title("hello world")            # "Hello World"
trimspace(" abc ")              # "abc"
replace("a-b-c", "-", "_")      # "a_b_c"
substr("abcdef", 0, 3)          # "abc"
format("%s-%03d", "web", 5)     # "web-005"
join(",", ["a", "b", "c"])      # "a,b,c"
split(",", "a,b,c")             # ["a","b","c"]

# collection
length([1,2,3])                 # 3
contains([1,2,3], 2)            # true
distinct([1,1,2,3])             # [1,2,3]
flatten([[1,2],[3,4]])          # [1,2,3,4]
merge({a=1}, {b=2})             # {a=1, b=2}
lookup({a=1}, "b", "default")   # "default"
keys({a=1, b=2})                # ["a","b"]
values({a=1, b=2})              # [1,2]
zipmap(["a","b"], [1,2])        # {a=1, b=2}
slice([1,2,3,4,5], 1, 3)        # [2,3]
sort(["c","a","b"])             # ["a","b","c"]

# 숫자
max(1,2,3)                      # 3
min(1,2,3)                      # 1
ceil(1.2)                       # 2
floor(1.8)                      # 1
abs(-5)                         # 5

# 인코딩
base64encode("hello")
base64decode("aGVsbG8=")
jsonencode({a=1, b="x"})        # JSON 문자열
jsondecode("{\"a\":1}")         # object
yamldecode(file("x.yaml"))

# 파일
file("config.txt")              # 파일 내용
filebase64("image.png")
templatefile("user-data.tpl", {var=value})

# 시간 / 날짜
timestamp()                     # "2026-05-15T..."
formatdate("YYYY-MM-DD", timestamp())
timeadd(timestamp(), "24h")

# 해시 / 암호화
sha256("hello")
md5("hello")
bcrypt("password")
filebase64sha256("file.zip")
```

---

## 6. cidr 관련

```hcl
cidrsubnet("10.0.0.0/16", 8, 1)    # "10.0.1.0/24" (subnet)
cidrhost("10.0.0.0/24", 5)         # "10.0.0.5"
cidrnetmask("10.0.0.0/24")         # "255.255.255.0"

# AZ 별 subnet 자동 생성
locals {
  azs = ["a", "b", "c"]
}

resource "aws_subnet" "private" {
  for_each = toset(local.azs)
  cidr_block = cidrsubnet(aws_vpc.main.cidr_block, 4, index(local.azs, each.key))
  availability_zone = "ap-northeast-2${each.key}"
}
```

---

## 7. count 의 index 문제 (★)

```hcl
# 문제: count
resource "aws_subnet" "old" {
  count      = 3
  cidr_block = cidrsubnet(...)
  ...
}

# 중간 삭제 시:
# subnet[0] subnet[1] subnet[2]
# → subnet[1] 삭제 want
# → terraform: index 2 → 1 으로 re-create (전체 영향)
```

```hcl
# 해결: for_each (key 기반)
resource "aws_subnet" "new" {
  for_each   = toset(["a", "b", "c"])
  cidr_block = cidrsubnet(...)
}

# "b" 삭제 → "a", "c" 영향 X
```

---

## 8. data source

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

resource "aws_instance" "web" {
  ami = data.aws_ami.ubuntu.id
}
```

```hcl
# 다른 stack 의 output 참조 (state)
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = {
    bucket = "my-tf-state"
    key    = "vpc/terraform.tfstate"
    region = "ap-northeast-2"
  }
}

resource "aws_instance" "web" {
  subnet_id = data.terraform_remote_state.vpc.outputs.subnet_ids[0]
}
```

---

## 9. lifecycle

```hcl
resource "aws_instance" "web" {
  # ...

  lifecycle {
    create_before_destroy = true       # 새 생성 후 옛 삭제 (zero downtime)
    prevent_destroy       = true       # destroy 시도 시 error
    ignore_changes        = [tags]     # tags 변경 무시 (manual edit OK)
    replace_triggered_by  = [
      aws_security_group.web.id        # SG 변경 시 instance recreate
    ]
  }
}
```

---

## 10. depends_on

```hcl
resource "aws_iam_role_policy" "policy" {
  # ...

  depends_on = [
    aws_iam_role.role,
    null_resource.wait_for_propagation
  ]
}
```

→ 자동 의존성 추론 안 되는 경우 명시.

---

## 11. moved block (★ terraform 1.1+)

```hcl
# 이름 변경
moved {
  from = aws_instance.web
  to   = aws_instance.application
}
```

→ rename 시 terraform 이 destroy + create 안 함. state 안에서만 이동.

---

## 12. 함정

1. **count 의 index 의존성** — for_each 권장.
2. **dynamic block over-use** — 가독성 ↓.
3. **timestamp() in resource** — apply 마다 변경 → drift.
4. **for / for_each 의 dependency** — 잘못된 순서.
5. **lifecycle ignore_changes 남용** — drift 누적.
6. **conditional 자원의 count = 0** — for_each 의 `{}` 가 더 명확.
7. **interpolation `${...}` legacy** — `var.x` 직접 사용 (terraform 0.12+).

---

## 13. 관련

- [[iac|↑ iac]]
- [[terraform-basics]]
- [[terraform-modules]]
- [[terraform-advanced-patterns]]
