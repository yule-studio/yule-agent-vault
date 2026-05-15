---
title: "IaC pitfalls"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:20:00+09:00
tags: [devops, iac, pitfalls]
---

# IaC pitfalls

**[[iac|↑ iac]]**

---

1. **state local commit** → 다중 user 충돌 + AK/SK leak.
2. **state lock 없음** → 동시 apply → corruption.
3. **provider version 미고정** → 다른 결과.
4. **count + reorder** → destroy/recreate (for_each 사용).
5. **secret tfvars commit** → vault / secret manager.
6. **module 자체 provider** → 호출자 override 어려움.
7. **terraform destroy prod** → prevent_destroy + workspace 분리.
8. **import 안 함** — 콘솔에서 만든 자원 → drift.
9. **manual cloud console 변경** → terraform plan 시 destroy 위협.
10. **plan 안 보고 apply** — `-auto-approve` 위험.
11. **scan 안 함** (tfsec / Checkov) → 보안 / 비용 누출.
12. **CIDR 충돌** — VPC peering 시.
13. **module 너무 깊음** — debug 어려움.
14. **CRD 누락** (Crossplane) → apply fail.
15. **stale state** — drift detection 없음.
16. **state file backup 없음** — corruption 시 복구 X.
17. **변수 type annotation X** → 잘못된 input 통과.
18. **tag 표준 없음** → cost allocation X.
19. **모든 env 같은 file** → dev 변경 시 prod 영향.
20. **destroy 의 dependency 순서** — 자원 간 의존 → 일부 fail.

---

## 관련

- [[iac|↑ iac]]
- [[best-practices]]
- [[terraform-state]]
