---
title: "Terraform troubleshooting — 자주 만나는 문제"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T09:46:00+09:00
tags: [devops, iac, terraform, troubleshooting]
---

# Terraform troubleshooting — 자주 만나는 문제

**[[iac|↑ iac]]**

---

## 1. 진단 도구

```bash
# verbose log
TF_LOG=DEBUG terraform apply         # TRACE / DEBUG / INFO / WARN / ERROR
TF_LOG=TRACE TF_LOG_PATH=tf.log terraform apply

# state inspect
terraform state list
terraform state show <resource>
terraform show
terraform output

# plan 분석
terraform plan -out=plan.tfplan
terraform show -json plan.tfplan > plan.json
jq '.resource_changes[]' plan.json
```

---

## 2. state lock 깨짐

```
Error: Error acquiring the state lock
Lock Info:
  ID:        ...
  Path:      ...
  Operation: OperationTypeApply
  Who:       alice@host
  Created:   2026-05-15 10:00:00
```

**원인:** 다른 apply / plan 진행 중 OR 이전 process 가 crash.

**해결:**
```bash
# 1. 정말 동시 실행 중인지 확인 (Slack / CI)
# 2. 아니면 force unlock
terraform force-unlock <LOCK_ID>

# DynamoDB 의 lock 직접 삭제 (last resort)
aws dynamodb delete-item \
    --table-name my-tf-locks \
    --key '{"LockID":{"S":"my-tf-state/prod/terraform.tfstate-md5"}}'
```

→ **force-unlock 신중**. 진짜 동시 실행이면 state 손상.

---

## 3. state drift (manual change)

```
누가 console 에서 instance type 변경 →
  terraform plan = "change in-place"
```

**선택:**
```bash
# A. terraform 의 가치 = code → 다시 정합 (drift 무시)
terraform apply           # 원래 값으로 복원

# B. 누군가의 변경 = 정당, 코드 update
# 코드의 instance_type 변경 → apply

# C. terraform 관리 빼기
terraform state rm aws_instance.web
```

```bash
# drift 정기 detect
terraform plan -refresh-only -detailed-exitcode
# exit code 2 = drift 있음
```

→ CI 매시간 drift 검출 + Slack alert.

---

## 4. resource already exists

```
Error: creating EC2 Instance: InvalidGroup.Duplicate: 
The security group 'web-sg' already exists
```

**원인:** 이미 cloud 에 자원 있음 (이전 apply / manual).

**해결:**
```bash
# A. import
terraform import aws_security_group.web sg-0abc

# B. 자원 삭제 후 재생성
aws ec2 delete-security-group --group-id sg-0abc
terraform apply
```

---

## 5. dependency cycle

```
Error: Cycle: 
  aws_instance.web → aws_security_group.web → aws_instance.web
```

**원인:** 두 자원이 서로 참조.

**해결:**
```hcl
# 분리
resource "aws_security_group" "web" {
  vpc_id = aws_vpc.main.id
}

resource "aws_security_group_rule" "web_ingress" {
  security_group_id = aws_security_group.web.id
  # ...
  source_security_group_id = aws_security_group.web.id  # 자기 자신 OK (별도 rule)
}
```

→ 자기 참조는 별도 resource 분리.

---

## 6. count / for_each 의 변경

```
- aws_subnet.private[0]
- aws_subnet.private[1]
- aws_subnet.private[2]

variable list 의 [0] 삭제 시:
  → terraform 은 마지막 element 삭제 (not [0])
  → 의도와 다름 = destroy + recreate
```

**해결:**
```hcl
# count 대신 for_each (key 명시)
resource "aws_subnet" "private" {
  for_each = {
    "a" = "10.0.1.0/24"
    "b" = "10.0.2.0/24"
    "c" = "10.0.3.0/24"
  }
  cidr_block = each.value
}

# "a" 만 삭제 시 — "b", "c" 영향 X
```

→ 이미 count 사용 중 + 변경 위험 = `moved` block 으로 rename.

---

## 7. provider version 충돌

```
Error: Failed to query available provider packages
Could not retrieve the list of available versions for provider hashicorp/aws
```

**해결:**
```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"          # 5.x 의 latest
    }
  }
}
```

```bash
terraform init -upgrade
```

→ `.terraform.lock.hcl` commit (lock).

---

## 8. apply 도중 실패

```
Error: ... 
Apply complete! Resources: 3 added, 0 changed, 1 destroyed.
But: some resources NOT applied due to error
```

**상태:** partial apply. state 에 일부만.

**해결:**
```bash
# 1. 현재 상태 확인
terraform state list
terraform plan         # 무엇이 빠졌는지

# 2. 다시 apply
terraform apply

# 3. orphan 자원 (state 에 없는데 cloud 에 있음) 정리
aws ec2 describe-instances --filters ...
```

→ retry 가능한 idempotent design.

---

## 9. resource taint (재생성 강제)

```bash
# 옛 명령 (deprecated)
terraform taint aws_instance.web

# 신 명령 (★)
terraform apply -replace=aws_instance.web

# 또는 코드에서
resource "aws_instance" "web" {
  lifecycle {
    replace_triggered_by = [aws_security_group.web.id]
  }
}
```

→ "이 자원만 새로 만들고 싶다".

---

## 10. backend migration

```
S3 → Terraform Cloud
또는 다른 S3 bucket 으로
```

```hcl
# 1. backend config 변경
terraform {
  backend "remote" {
    organization = "my-org"
    workspaces {
      name = "prod"
    }
  }
}

# 2. migrate (data 이동 자동)
terraform init -migrate-state

# 3. yes 확인
```

→ 위험. backup 후 시도.

---

## 11. corrupted state

```
Error: state file is corrupted
```

**해결:**
```bash
# 1. S3 versioning 활용 — 이전 version 으로 복원
aws s3api list-object-versions --bucket my-tf-state --prefix prod/
aws s3api copy-object --bucket my-tf-state \
    --copy-source "my-tf-state/prod/terraform.tfstate?versionId=v123" \
    --key prod/terraform.tfstate

# 2. local backup 사용
cp terraform.tfstate.backup terraform.tfstate

# 3. 그래도 안 되면 — 새 state, terraform import 로 재구축
```

→ **S3 versioning 항상 켜기**.

---

## 12. plan output 너무 많음

```bash
# 특정 resource 만 plan
terraform plan -target=aws_instance.web

# JSON 출력 + jq
terraform show -json plan.tfplan | jq '.resource_changes[] | select(.change.actions[] == "delete")'

# diff 만 (구조 변화)
terraform plan -no-color | grep -E "^[+-]"
```

→ `-target` 은 trouble shooting / 응급용. 일반 사용 X.

---

## 13. 자주 보는 error

| | 원인 | 해결 |
| --- | --- | --- |
| InvalidParameterValue | cloud API 거부 | manifest 검토 |
| ResourceNotFoundException | data source 가 못 찾음 | filter 검토 |
| UnauthorizedOperation | IAM 권한 | policy 추가 |
| RequestExpired | clock drift | NTP |
| ProvisionedThroughputExceeded | DynamoDB lock 한계 | retry / capacity ↑ |
| ResourceInUseException | 다른 자원이 참조 | dependency 먼저 삭제 |
| InvalidVpcID.NotFound | VPC 가 cross-region | provider region |
| AlreadyExists | 자원 충돌 | import 또는 삭제 |

---

## 14. 함정

1. **`-target` 일상 사용** — drift 누적.
2. **state file local + git commit** — secret 노출.
3. **force-unlock 남용** — state 손상.
4. **destroy → 처음부터** — backup 없이 prod.
5. **plan output 무시 + apply** — 실수 자원 변경.
6. **-auto-approve in prod** — review 없이.
7. **manual API call + terraform** — drift / 충돌.

---

## 15. 관련

- [[iac|↑ iac]]
- [[terraform-state]]
- [[terraform-testing]]
- [[pitfalls]]
