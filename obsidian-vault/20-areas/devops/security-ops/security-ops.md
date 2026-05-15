---
title: "Security Operations — secrets / SIEM / zero trust / compliance"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T08:00:00+09:00
tags: [area, devops, security-ops]
---

# Security Operations — secrets / SIEM / zero trust / compliance

**[[../devops|↑ devops]]**

---

## 1. 영역

DevOps 운영의 보안 측면:

```
[infra] ─ access control ─→ [user]
   │
   ├─ secrets management
   ├─ identity (SSO / MFA)
   ├─ audit log (SIEM)
   ├─ compliance (SOC2 / ISO27001 / PCI-DSS / HIPAA / GDPR)
   ├─ vulnerability mgmt (CVE 추적)
   ├─ supply chain (SBOM, image signing)
   └─ incident response (보안 incident)
```

---

## 2. 하위 영역

- [[secrets-management]] — Vault / AWS Secrets Manager / SOPS / ESO
- [[iam-best-practices]] — least privilege / IAM / RBAC
- [[zero-trust]] — BeyondCorp / mTLS / SPIFFE
- [[siem]] — Splunk / Elastic SIEM / Wazuh / Datadog Security
- [[vulnerability-management]] — CVE / Snyk / Trivy / Dependabot
- [[supply-chain-security]] — SBOM / cosign / SLSA / OIDC for CI
- [[compliance]] — SOC2 / ISO27001 / PCI-DSS / GDPR / HIPAA
- [[threat-modeling]] — STRIDE / DREAD
- [[security-incident]] — 침해 사고 대응
- [[pitfalls]]

---

## 3. 책임 분담 (shared responsibility)

```
[Cloud Provider (AWS/GCP)]
  - hardware / hypervisor / network / data center

[조직]
  - OS patch (managed service 외)
  - application 보안
  - data 보호 / 암호화
  - IAM / 권한
  - secrets
  - compliance
```

→ "cloud = 자동 안전" 오해 ❌.

---

## 4. CIA triad

| | 무엇 |
| --- | --- |
| **Confidentiality** | 비공개 — 권한 있는 자만 접근 |
| **Integrity** | 무결성 — 변조 X |
| **Availability** | 가용성 — 필요할 때 사용 |

→ 보안 = 3 모두 보장. DDoS = availability 공격.

---

## 5. Defense in Depth

```
사용자 → CDN/WAF → LB → API GW → Ingress → Mesh → Pod (app input validation) → DB (least priv)
        ↑          ↑       ↑          ↑          ↑       ↑                      ↑
        DDoS      WAF     auth      TLS         mTLS    SQL injection          encryption
```

→ 한 layer fail 해도 다음 layer.

---

## 6. 핵심 원칙

1. **least privilege** — 최소 권한.
2. **zero trust** — 항상 검증 (내부도).
3. **defense in depth** — 다층.
4. **fail safe** — 실패 시 deny.
5. **separation of duties** — 한 사람이 모두 못 함.
6. **audit everything** — 누가 무엇.
7. **encryption at rest + in transit** — 둘 다.
8. **secrets management** — 코드 / config 에 X.
9. **patch / update** — 즉시.
10. **assume breach** — 침해 가정 — 탐지 / 회복.

---

## 7. 학습 순서

1. Day 1 — [[secrets-management]], [[iam-best-practices]]
2. Day 2 — [[zero-trust]] + [[../networking-ops/network-policy|NetworkPolicy]]
3. Day 3 — [[vulnerability-management]] + [[supply-chain-security]]
4. Day 4 — [[siem]] + [[security-incident]]
5. Day 5 — [[compliance]] + [[threat-modeling]]

---

## 8. 관련

- [[../devops|↑ devops]]
- [[../networking-ops/waf|↗ WAF]]
- [[../networking-ops/network-policy|↗ Network Policy]]
- [[../docker/security|↗ docker security]]
- [[../kubernetes/rbac|↗ k8s RBAC]]
