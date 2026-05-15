---
title: "IaC 도구 비교 ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:02:00+09:00
tags: [devops, iac, comparison]
---

# IaC 도구 비교 ★

**[[iac|↑ iac]]**

---

## 1. 매트릭스

| | **Terraform** ★ | Pulumi | CDK | CloudFormation | Crossplane | Ansible |
| --- | --- | --- | --- | --- | --- | --- |
| 언어 | HCL | TS/Py/Go/.NET | TS/Py/Java/.NET | JSON/YAML | YAML (CRD) | YAML |
| state | S3 + DynamoDB | service / self | CFN stack | AWS internal | k8s etcd | 없음 |
| Cloud | 1000+ provider | 1000+ | AWS / CDKtf / CDK8s | AWS only | k8s + cloud | 무한 |
| 학습 | 쉬움 | 약간 ↑ | AWS 의존 ↑ | 어려움 | k8s 의존 | 쉬움 |
| 가격 | OSS (Cloud $) | OSS (Cloud $) | OSS | OSS | OSS | OSS |
| typed | X | O | O | X | X | X |
| GitOps | OK | OK | OK | OK | 강 (k8s) | OK |
| 서버 config | X | X | X | X | X | O |

---

## 2. 어떤 거 언제

| 상황 | 추천 |
| --- | --- |
| **시작 / 다국 cloud** | Terraform ★ |
| **AWS 전용 + TS** | CDK |
| **typed 필수 (큰 팀)** | Pulumi |
| **k8s-native** | Crossplane |
| **VM 설정 (apt install / config 변경)** | Ansible |
| **dev experience 우선** | Pulumi |
| **legacy CloudFormation** | 그대로 (관성) |

---

## 3. Terraform 의 강점

- ✓ provider 가장 많음 (1000+)
- ✓ HCL 가독성
- ✓ community / 자료
- ✗ HCL DSL 한계 (real loop / complex logic)
- ✗ state 의 manual 관리

---

## 4. Pulumi 의 강점

- ✓ TS / Python / Go 등 real language
- ✓ IDE autocomplete / type check
- ✓ test (jest / pytest)
- ✗ state 가 Pulumi Service 의존 (또는 self-host)
- ✗ HCL 보다 자료 적음

---

## 5. CDK 의 강점

- ✓ AWS 깊은 통합 (L1/L2/L3 construct)
- ✓ TypeScript IDE
- ✗ AWS 전용 (CDK8s / CDKtf 별도)

---

## 6. Crossplane

```yaml
# k8s manifest 로 cloud 리소스 정의
apiVersion: rds.aws.upbound.io/v1beta1
kind: Instance
metadata: {name: prod-db}
spec:
  forProvider:
    region: ap-northeast-2
    instanceClass: db.t3.medium
    engine: postgres
```

→ ArgoCD + Crossplane = "k8s manifest 하나로 cloud + app".

---

## 7. Ansible vs Terraform

| | Ansible | Terraform |
| --- | --- | --- |
| state | X (idempotent commands) | O |
| 사용 | 서버 config (apt / sed / service) | 인프라 자원 (VM / DB / VPC) |
| 둘 다? | O (Terraform → VM, Ansible → 안에서 config) |

---

## 8. 관련

- [[iac|↑ iac]]
- [[terraform-basics]]
- [[pulumi]]
- [[ansible]]
- [[crossplane]]
