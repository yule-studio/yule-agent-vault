---
title: "GCP Security / IAM (Hub)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:25:00+09:00
tags:
  - gcp
  - security
  - iam
  - hub
---

# GCP Security / IAM (Hub)

**[[../cloud-gcp|↑ GCP]]**

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **IAM** | [[iam]] | role / service account / workload identity |
| **Secret Manager** | [[secret-manager]] | API key / DB password |
| **Cloud KMS** | [[cloud-kms]] | 암호화 key |

추가:
- **Cloud Armor** — WAF
- **Identity-Aware Proxy (IAP)** — Zero Trust access
- **Security Command Center** — SCC (Wiz / CSPM 동등)
- **VPC Service Controls** — data exfiltration 방지

---

## AWS 매핑

| AWS | GCP |
| --- | --- |
| IAM | IAM |
| KMS | Cloud KMS |
| Secrets Manager | Secret Manager |
| WAF | Cloud Armor |
| Cognito | Identity Platform (Firebase Auth) |
| GuardDuty | Security Command Center |

---

## 관련

- [[../cloud-gcp|↑ GCP]]
