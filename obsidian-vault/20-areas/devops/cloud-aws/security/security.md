---
title: "AWS Security / IAM (Hub)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:09:00+09:00
tags:
  - aws
  - security
  - iam
  - hub
---

# AWS Security / IAM (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 카테고리 hub |

**[[../cloud-aws|↑ AWS]]**

---

## 1. 서비스

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **IAM** | [[iam]] | 인증 / 인가의 토대 |
| **KMS** | [[kms]] | Key Management — 암호화 키 |
| **Secrets Manager** | [[secrets-manager]] | DB password / API token 안전 보관 |

---

## 2. 보안 원칙

1. **Least Privilege** — 필요한 최소 권한만
2. **Defense in Depth** — IAM + Security Group + NACL + WAF
3. **Encryption at Rest + In Transit**
4. **Audit** — CloudTrail (모든 API 호출 log)
5. **MFA** — root + admin 사용자 필수
6. **No Shared Credentials** — User / Role 각자

---

## 3. 추가 서비스 (별도 노트 예정)

- **WAF** — Web ACL (SQL injection / XSS / rate limit)
- **Shield** — DDoS
- **GuardDuty** — 이상 탐지
- **Macie** — S3 민감 데이터 검출
- **Cognito** — 사용자 인증 (login / OAuth)
- **ACM** — TLS 인증서 (무료)
- **Inspector** — 취약점 스캔

---

## 4. 관련

- [[../cloud-aws|↑ AWS]]
- [[../../../computer-science/security-theory/security-theory|↗ 보안 이론]]
- [[../../security-ops/security-ops]]
