---
title: "Terraform testing — terratest / native test / TFLint"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T09:40:00+09:00
tags: [devops, iac, terraform, testing]
---

# Terraform testing — terratest / native test / TFLint

**[[iac|↑ iac]]**

---

## 1. 왜

```
IaC 도 코드 → 검증 필요:
  - syntax error
  - bug (잘못된 cidr / 잘못된 IAM policy)
  - security (public S3 bucket)
  - drift (manual change 가 적용되지 않을지)
  - regression (옛 working config 가 새 변경으로 깨짐)
```

→ "terraform apply 후 prod 죽었다" 방지.

---

## 2. testing 종류 (피라미드)

```
       ┌──────────────────┐
       │  manual (smoke)  │   ← 적게
       ├──────────────────┤
       │  e2e / terratest │   ← 중간
       ├──────────────────┤
       │  integration     │
       ├──────────────────┤
       │  unit            │   ← 많이
       └──────────────────┘

unit:        terraform validate, TFLint, tfsec, checkov
integration: terraform plan + 검증 (terraform-compliance)
e2e:         terratest (실제 apply + 검증 + destroy)
```

---

## 3. terraform validate (★ 무료)

```bash
terraform validate

# CI 에서 항상
terraform fmt -check -recursive   # 포맷 검증
terraform validate                 # syntax + provider 자료형
```

→ syntax error 만. 의미 정확성 X.

---

## 4. TFLint (★)

```bash
brew install tflint

# 설정
cat > .tflint.hcl <<EOF
plugin "aws" {
    enabled = true
    version = "0.30.0"
    source  = "github.com/terraform-linter/tflint-ruleset-aws"
}

config {
    module = true
}

rule "terraform_required_version" {
    enabled = true
}

rule "terraform_required_providers" {
    enabled = true
}
EOF

tflint --init
tflint --recursive
```

검출:
- deprecated argument
- invalid AWS instance type
- missing required provider
- naming convention

---

## 5. tfsec / checkov (★ 보안 / 정책)

```bash
# tfsec
brew install tfsec
tfsec .

# 또는 trivy 의 IaC
trivy config .

# checkov (★ 더 풍부)
pip install checkov
checkov -d .
checkov -f main.tf --framework terraform

# 특정 check 만
checkov -d . --check CKV_AWS_18,CKV_AWS_57

# 무시
# main.tf 안에:
# checkov:skip=CKV_AWS_18:이유 설명
resource "aws_s3_bucket" "logs" {
    # ...
}
```

검출:
- public S3 bucket
- SSH from 0.0.0.0/0
- IAM `*` permission
- encryption 누락
- backup 정책 누락

---

## 6. terraform-compliance (BDD style)

```bash
pip install terraform-compliance
```

```gherkin
# features/security.feature
Feature: Security checks

  Scenario: Ensure S3 buckets have encryption
    Given I have aws_s3_bucket defined
    Then it must contain server_side_encryption_configuration

  Scenario: No public SSH
    Given I have aws_security_group defined
    When it contains ingress
    Then its from_port must not be 22
      Or its cidr_blocks must not contain "0.0.0.0/0"
```

```bash
terraform plan -out=plan.out
terraform show -json plan.out > plan.json
terraform-compliance -f features/ -p plan.json
```

→ 자연어 가까운 정책. compliance team 이 작성 가능.

---

## 7. terraform native test (★ 1.6+)

```hcl
# tests/vpc.tftest.hcl
run "create_vpc" {
  command = plan

  variables {
    cidr = "10.0.0.0/16"
    env  = "test"
  }

  assert {
    condition     = aws_vpc.main.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR mismatch"
  }

  assert {
    condition     = length([for s in aws_subnet.private : s.id]) == 3
    error_message = "Should have 3 private subnets"
  }
}

run "apply_and_check" {
  command = apply

  assert {
    condition     = output.vpc_id != ""
    error_message = "VPC ID should not be empty"
  }
}
```

```bash
terraform test
```

→ 내장. 외부 도구 불필요. terraform 1.6+ 표준.

---

## 8. terratest (★ Go, e2e)

```go
// vpc_test.go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/gruntwork-io/terratest/modules/aws"
    "github.com/stretchr/testify/assert"
)

func TestVpcCreate(t *testing.T) {
    t.Parallel()

    terraformOptions := &terraform.Options{
        TerraformDir: "../examples/vpc",
        Vars: map[string]interface{}{
            "cidr": "10.99.0.0/16",
            "env":  "test-" + random.UniqueId(),
        },
    }

    defer terraform.Destroy(t, terraformOptions)   // 끝나면 정리

    terraform.InitAndApply(t, terraformOptions)

    vpcId := terraform.Output(t, terraformOptions, "vpc_id")
    region := terraform.Output(t, terraformOptions, "region")

    // 실제 AWS API 호출 검증
    vpc := aws.GetVpcById(t, vpcId, region)
    assert.Equal(t, "10.99.0.0/16", *vpc.CidrBlock)
    assert.Len(t, vpc.Tags, 2)
}
```

```bash
go test -v -timeout 30m ./...
```

→ 실제 AWS 자원 만들고 검증 + 정리. 비용 발생 (CI 안에서 sandbox account 권장).

---

## 9. mock provider (★ 1.7+)

```hcl
# mock 으로 외부 호출 없이 test
mock_provider "aws" {
  mock_resource "aws_instance" {
    defaults = {
      id          = "i-mockid"
      arn         = "arn:aws:..."
      public_ip   = "1.2.3.4"
    }
  }
}

run "verify" {
  command = plan
  assert {
    condition     = aws_instance.web.public_ip == "1.2.3.4"
    error_message = "..."
  }
}
```

→ 빠름 (network 호출 X). 큰 코드 회귀 test 적합.

---

## 10. CI pipeline 예 (GitHub Actions)

```yaml
name: terraform
on: [pull_request]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform fmt -check -recursive
      - run: terraform init -backend=false
      - run: terraform validate
      - uses: terraform-linters/setup-tflint@v4
      - run: tflint --recursive
      - run: |
          curl -s https://raw.githubusercontent.com/aquasecurity/tfsec/master/scripts/install_linux.sh | bash
          tfsec .

  plan:
    runs-on: ubuntu-latest
    needs: validate
    permissions: {id-token: write}      # OIDC
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123:role/terraform-plan
          aws-region: ap-northeast-2
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init
      - run: terraform plan -out=plan.tfplan
      - run: terraform-compliance -f features/ -p plan.tfplan
      - run: terraform test

  e2e:
    runs-on: ubuntu-latest
    needs: plan
    if: github.event_name == 'workflow_dispatch'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: {go-version: '1.21'}
      - run: cd test && go test -v -timeout 60m
```

---

## 11. golden file test

```bash
# 첫 plan → golden
terraform plan -out=plan
terraform show -json plan > tests/golden/vpc.json

# 다음 commit 시 비교
terraform plan -out=plan-new
terraform show -json plan-new > current.json
diff tests/golden/vpc.json current.json
```

→ 의도치 않은 변경 검출.

---

## 12. 함정

1. **e2e test 실제 production account** — sandbox account 필수.
2. **e2e cleanup 안 함** — orphan resource 비용.
3. **secret in test** — Vault / env var.
4. **test 의 random name 충돌** — uniqueId / timestamp.
5. **mock 만 — 실제 cloud 호환 검증 X** — e2e 도 필요.
6. **policy as code (tfsec/checkov) noise** — 점진 enforce.
7. **test 만, plan review 안 함** — 자동화 + 사람 review 같이.

---

## 13. 관련

- [[iac|↑ iac]]
- [[terraform-basics]]
- [[../security-ops/vulnerability-management|↗ vuln mgmt]]
- [[../cicd/pipeline-patterns|↗ CI/CD]]
