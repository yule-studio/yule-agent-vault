---
title: "product security hub"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:55:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - security
---

# product security hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[../product|↑ hub]]**

---

## 1. 영역

| 영역 | 내용 | 노트 |
| --- | --- | --- |
| PCI-DSS | 카드번호 / CVC 서버 저장 X | [[pci-dss]] |
| Webhook 서명 | PG webhook HMAC 검증 | [[webhook-signature]] |
| Idempotency | 결제 중복 방어 | [[idempotency-key]] |
| 디지털 워터마크 | PDF 마스킹 + 토큰 | [[digital-watermarking]] |
| PII 암호화 | 배송지 column-level | [[pii-encryption]] |
| Audit | admin / system action 5년 | [[audit-logging]] |

---

## 2. signup security 와의 관계

- 인증 (JWT) = signup 의 것 그대로 ([[../../signup/security/security|↗]]).
- product 는 인증 위의 추가 보안.

---

## 3. 관련

- [[../product|↑ hub]]
- [[../../signup/security/security|↗ signup security]]
