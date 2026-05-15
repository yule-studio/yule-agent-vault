---
title: "IaC — Terraform / Pulumi / CDK / Ansible"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:00:00+09:00
tags: [devops, iac, terraform, hub]
---

# IaC — Terraform / Pulumi / CDK / Ansible

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[../devops|↑ devops]]**

---

## 0. 왜 IaC

| 질문 | 답 |
| --- | --- |
| 왜 IaC? | 클릭 운영 → 재현 X. 코드 = git 추적 + review + rollback |
| 왜 Terraform? | de facto 표준, 다국 cloud, HCL 가독성 |
| 왜 Pulumi / CDK? | 진짜 언어 (TS / Python / Go) — typed |
| 왜 Ansible? | 서버 config (SSH 기반) — IaC + 설정 관리 |

---

## 1. 영역

| 노트 | 내용 |
| --- | --- |
| [[tools-comparison]] ★ | Terraform / Pulumi / CDK / Crossplane / Ansible |
| [[terraform-basics]] ★ | provider / resource / state |
| [[terraform-modules]] | 재사용 패턴 |
| [[terraform-state]] | remote backend + locking |
| [[pulumi]] | typed language |
| [[aws-cdk]] | AWS 전용 |
| [[ansible]] | 서버 config |
| [[crossplane]] | k8s native IaC |
| [[best-practices]] | layout / env / secret |
| [[pitfalls]] | 흔한 함정 |
| [[practice/practice]] ★ | 실습 |

---

## 2. 도구 비교 요약

| | Terraform | Pulumi | CDK | Ansible |
| --- | --- | --- | --- | --- |
| 언어 | HCL | TS/Py/Go/.NET | TS/Py/Java/.NET | YAML |
| state | external (s3) | service or self | CloudFormation | none |
| provider | 1000+ | 1000+ | AWS only (CDK), gcp/azure 별도 | 무한 (모듈) |
| 학습 | 쉬움 | 약간 ↑ | AWS 의존 | 쉬움 |
| 본 vault | ★ | option | AWS only | 옵션 (VM) |

자세히: [[tools-comparison]].

---

## 3. 본 vault 권장

```
시작:
  Terraform + 1 cloud (AWS / GCP / Azure)

성장:
  + 모듈 (network / iam / db 분리)
  + remote state (S3 + DynamoDB lock)
  + tfsec / Checkov scan

대형:
  + Atlantis / Terraform Cloud (PR plan/apply)
  + Crossplane (k8s-native)
  + 또는 Pulumi (typed)
```

---

## 4. 관련

- [[../devops|↑ devops]]
- [[../kubernetes/kubernetes|↗ k8s]]
- [[../cloud-aws/cloud-aws|↗ AWS]]
- [[../cicd/cicd|↗ cicd]]
