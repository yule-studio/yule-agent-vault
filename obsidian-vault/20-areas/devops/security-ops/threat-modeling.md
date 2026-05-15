---
title: "Threat modeling — STRIDE / DREAD"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T08:16:00+09:00
tags: [devops, security-ops, threat-modeling]
---

# Threat modeling — STRIDE / DREAD

**[[security-ops|↑ security-ops]]**

---

## 1. 무엇

design / architecture 단계에서 **위협을 체계적으로 식별**.

> "What can go wrong?"

→ 코드 작성 후 fix 가 아니라 **사전 예방**. Shift left security.

---

## 2. 절차 (4 question)

```
1. What are we building?     — 시스템 다이어그램 (DFD)
2. What can go wrong?        — STRIDE / kill chain
3. What are we doing?        — 현재 mitigation
4. Did we do a good job?     — review / iterate
```

---

## 3. STRIDE (Microsoft)

| | 무엇 | 위반 |
| --- | --- | --- |
| **S** poofing | 정체 위조 | Authentication |
| **T** ampering | 데이터 변조 | Integrity |
| **R** epudiation | 거부 (안 했다 주장) | Non-repudiation |
| **I** nformation Disclosure | 누출 | Confidentiality |
| **D** enial of Service | 서비스 거부 | Availability |
| **E** levation of Privilege | 권한 상승 | Authorization |

→ DFD 의 각 component 마다 STRIDE 6 가지 점검.

---

## 4. DFD 예 (로그인 시스템)

```
[user] ──[1]──→ [Web]
                  │
                 [2]
                  ↓
                [Auth Service] ──[3]──→ [DB (passwords)]
                  │
                 [4]
                  ↓
                [Audit log]
```

각 화살표 / box 별:

| 요소 | STRIDE 위협 |
| --- | --- |
| [1] user → Web | S: session hijack / S: phishing / T: MITM |
| Web | S: XSS / T: client-side / I: cookie |
| [2] Web → Auth | T: header 위조 / S: 다른 서비스 흉내 |
| Auth Service | S: brute-force / E: SQL injection bypass |
| [3] Auth → DB | I: 평문 password / T: DB 변조 |
| DB | I: backup 탈취 / E: SQL injection / D: query timeout |
| Audit | R: log 삭제 / T: log 변조 |

각 위협 → mitigation:
- session: SameSite cookie + secure
- brute-force: rate-limit + account lock
- password: bcrypt + salt
- audit: append-only / 분리 storage
- ...

---

## 5. DREAD (영향 score)

```
Damage          (피해 크기)        0-10
Reproducibility (재현성)            0-10
Exploitability  (공격 난이도)        0-10
Affected users  (영향 범위)          0-10
Discoverability (발견 난이도)         0-10
```

각 위협 score 합산 → 우선순위.

→ 주관적이라 현재는 CVSS 더 표준.

---

## 6. CIA / DIE / PASTA 기타

```
CIA — Confidentiality / Integrity / Availability (위반 시 영향)
DIE — Distributed / Immutable / Ephemeral (cloud 친화)
PASTA — Process for Attack Simulation and Threat Analysis (7 stage)
LINDDUN — privacy 중심
```

---

## 7. 도구

| | 무엇 |
| --- | --- |
| **MS Threat Modeling Tool** | Microsoft 무료 |
| **OWASP Threat Dragon** | OSS |
| **IriusRisk** | enterprise SaaS |
| **draw.io / mermaid** | DFD 작성 |
| **Pytm** | code 로 threat model |

```python
# Pytm 예
from pytm import TM, Actor, Boundary, Server, Datastore, Dataflow

tm = TM("My App Threat Model")
internet = Boundary("Internet")
company = Boundary("Company Network")

user = Actor("User", inBoundary=internet)
web = Server("Web Server", inBoundary=company)
db = Datastore("Database", inBoundary=company)

Dataflow(user, web, "HTTPS")
Dataflow(web, db, "JDBC")

tm.process()  # → STRIDE 자동 생성
```

---

## 8. 언제 / 어떻게 (회사 도입)

```
- 신규 service 의 design review 시
- 큰 architecture 변경 시
- compliance audit 준비
- security incident 후

빈도:
  - 큰 신규 = 처음 1회 깊게
  - 정기 = 분기 점검
  - threat 신규 발견 시 update
```

→ 1 시간 워크숍 (architect + dev + security + product).

---

## 9. mitigation 분류 (★)

| 전략 | 무엇 |
| --- | --- |
| **avoid** | 위험한 기능 제거 |
| **mitigate** | control 으로 영향 ↓ |
| **transfer** | 보험 / SaaS 위탁 |
| **accept** | 비용 > 영향, 의식적 허용 |

→ "accept" 도 합법 — 단, 명시적 결정 + 문서화.

---

## 10. 실전 예 — 결제 시스템

```
Threat: 카드번호 (PAN) 탈취
  → mitigation: PAN 저장 X, PG 사용 (tokenize)
  → mitigation: TLS 1.2+ + HSTS + CSP

Threat: 중복 결제 (replay)
  → mitigation: idempotency key (uuid) + DB unique index

Threat: webhook 위조 (PG 가 아닌 사람이 호출)
  → mitigation: HMAC signature 검증

Threat: refund 권한 escalation
  → mitigation: refund 는 별도 권한 + 2-person approval > $100

Threat: race condition (재고 부족 시 중복)
  → mitigation: SELECT FOR UPDATE 또는 Redis distributed lock
```

→ [[../../60-recipes/spring/product/payment/payment|payment recipe]] 의 보안 결정의 근거.

---

## 11. 함정

1. **design 후에야 시작** — fix 비쌈.
2. **너무 자세 → 막대한 시간** — 핵심 component 만.
3. **개발팀 참여 없이 security 만** — 현실성 ↓.
4. **document 만, 적용 X** — control 검증.
5. **위협 추적 안 함** — 같은 위협 다음 service 에서 반복.

---

## 12. 관련

- [[security-ops|↑ security-ops]]
- [[security-incident]]
- [[iam-best-practices]]
- [[../../60-recipes/spring/product/payment/payment|↗ payment 보안]]
