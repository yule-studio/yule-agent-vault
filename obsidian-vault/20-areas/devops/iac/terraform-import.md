---
title: "Terraform import — 기존 인프라 가져오기"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T09:44:00+09:00
tags: [devops, iac, terraform, import]
---

# Terraform import — 기존 인프라 가져오기

**[[iac|↑ iac]]**

---

## 1. 왜

```
이미 있는 자원 (console / CLI 로 만든) →
  Terraform 으로 관리 시작

이유:
  - legacy → IaC 마이그레이션
  - manual 자원을 코드화
  - 새 팀이 인수
  - 회사 표준이 IaC 만
```

→ "기존 자원 = 새로 만들면 downtime" → import.

---

## 2. classic terraform import (CLI)

```bash
# 1. 코드 먼저 작성 (빈 resource block)
cat > main.tf <<EOF
resource "aws_instance" "web" {
  # 정확한 config 모를 시 — 우선 빈 block
}
EOF

# 2. import
terraform import aws_instance.web i-0abc1234

# 3. state 에 자원 정보 들어옴
terraform state show aws_instance.web

# 4. show 의 결과 보고 main.tf 채움
# 모든 필드를 코드로 옮김

# 5. plan = no changes
terraform plan
```

→ 단점: 빈 block + manual 채움. 큰 자원 = 시간 많이.

---

## 3. import block (★ terraform 1.5+)

```hcl
# main.tf
import {
  to = aws_instance.web
  id = "i-0abc1234"
}

resource "aws_instance" "web" {
  ami           = "ami-..."
  instance_type = "t3.micro"
  # 정확한 config
}
```

```bash
terraform plan -generate-config-out=generated.tf
# → generated.tf 에 자동 코드 생성

# 코드 검토 후
terraform apply
```

→ 코드 자동 생성 + import. **현재 표준**.

---

## 4. terraformer (★ bulk)

```bash
# AWS 전체 region scan
terraformer import aws \
    --resources=vpc,subnet,sg,ec2_instance,rds \
    --regions=ap-northeast-2 \
    --profile=prod

# 결과
ls generated/aws/
# 각 resource 의 .tf 파일 + state
```

→ 수백 자원 일괄 import. 시작점 — 그 후 코드 정리.

---

## 5. import 절차 (★ 권장 flow)

```
1. 영향 자원 목록 작성
   aws ec2 describe-vpcs / instances / ...

2. terraform module 디자인 (어떻게 추상화)

3. import block 으로 한 자원씩
   terraform plan -generate-config-out=imported.tf

4. 코드 검토 / refactor
   - module 로 묶기
   - variable 추출

5. terraform plan → no changes

6. 그 다음 자원 반복

7. 모든 자원 import 후 commit

8. 새 자원은 terraform apply 로만
```

→ 한 번에 다 하지 말고 점진. 검증 가능 단위.

---

## 6. data source vs import

```hcl
# 자원 관리 X — 단순 참조
data "aws_vpc" "existing" {
  filter {
    name   = "tag:Name"
    values = ["prod-vpc"]
  }
}

resource "aws_subnet" "new" {
  vpc_id = data.aws_vpc.existing.id   # 참조만
}
```

```hcl
# 자원 관리 — terraform 이 변경 / destroy 가능
import {
  to = aws_vpc.existing
  id = "vpc-0abc"
}

resource "aws_vpc" "existing" {
  cidr_block = "10.0.0.0/16"
  tags = {Name = "prod-vpc"}
}
```

→ 변경 권한 필요 = import. 단순 참조 = data source.

---

## 7. 분리된 자원

```
한 EC2 instance + EBS volume + ENI + Elastic IP + SG:
  각각 별도 resource → 각각 import 필요

aws_instance.web                — instance
aws_ebs_volume.web_data         — EBS
aws_eip.web                     — EIP
aws_eip_association.web         — association
aws_network_interface.web       — ENI
aws_security_group.web          — SG
```

→ AWS console 의 한 instance 가 terraform 에서 6+ resource.

---

## 8. tag 기반 import

```bash
# 1. tag 가 같은 모든 자원 검색
aws ec2 describe-instances \
    --filters "Name=tag:ManagedBy,Values=manual" \
    --query 'Reservations[].Instances[].InstanceId' \
    --output text

# 2. 각 결과 import
for id in $instances; do
    cat >> imports.tf <<EOF
import {
  to = aws_instance.imported_${id}
  id = "${id}"
}
EOF
done

# 3. generate
terraform plan -generate-config-out=imported.tf
```

---

## 9. state rm vs import

```bash
# 자원은 cloud 에 두고 state 에서만 제거 (terraform 관리 X)
terraform state rm aws_instance.web

# 반대
terraform import aws_instance.web i-0abc
```

→ rm 후 cloud 의 자원은 그대로. import 로 다시 가져올 수 있음.

→ 다른 stack 으로 자원 이동 시:
```bash
# stack A 에서 rm
terraform state rm aws_instance.web

# stack B 에서 import
terraform import aws_instance.web i-0abc
```

---

## 10. 함정

1. **import 후 plan 의 변경** — 코드 ≠ 실제 상태. 코드 fix.
2. **tag 누락 후 import 강행** — 다음 apply 시 tag 추가 (drift).
3. **lifecycle ignore_changes** 남용 — drift 인식 X.
4. **terraformer 결과 그대로** — 정리 없이 commit 하면 카오스.
5. **분리된 자원 일부만 import** — 부분 관리 → 충돌.
6. **import 와 동시 apply** — race. 한 명이 import.
7. **state lock 없이 동시 import** — 손상 가능.

---

## 11. tips

```bash
# 도구
- terraformer (Google) — bulk
- former2 — AWS console UI 로 import (web)
- aws2tf — AWS → terraform 변환
- pulumi convert — Pulumi 의 변환 (terraform 도)

# 검증
terraform plan -refresh-only      # state vs cloud 비교 (apply X)
terraform refresh                  # state 만 sync
```

---

## 12. 관련

- [[iac|↑ iac]]
- [[terraform-state]]
- [[terraform-basics]]
- [[terraform-advanced-patterns]]
