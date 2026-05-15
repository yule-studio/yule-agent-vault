---
title: "IDP — Internal Developer Platform 개념"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T08:24:00+09:00
tags: [devops, platform-engineering, idp]
---

# IDP — Internal Developer Platform 개념

**[[platform-engineering|↑ platform-engineering]]**

---

## 1. cognitive load 줄이기 (★)

개발자가 알아야 할 것:
```
순수 개발자 cognitive load:
  - 자기 domain (business logic)
  - 언어 / framework
  - 라이브러리
  - testing
  → 충분

추가 부담:
  - Docker
  - k8s
  - Helm
  - CI/CD (Actions/Jenkins)
  - IaC (Terraform)
  - cloud (AWS / GCP)
  - monitoring (Prometheus / Grafana)
  - security (IAM / secrets)
  - DR / backup
  → 과중. learning curve 6개월+

→ Platform 이 이 부담 흡수.
```

→ Platform = **abstraction over infrastructure complexity**.

---

## 2. 5 plane (Humanitec / CNCF 모델)

```
[Developer Control plane]    — UI / CLI / git (Backstage, Port)
[Integration & Delivery]     — CI/CD (GitHub Actions, ArgoCD)
[Monitoring & Logging]       — Prometheus, Loki, Tempo
[Security]                   — Vault, secrets, OPA
[Resource]                   — k8s, cloud (Crossplane, Terraform)
```

각 plane 에 책임 분리.

---

## 3. user persona

| | 무엇 |
| --- | --- |
| **Application Developer** | 일반 개발자, IDP 의 main user |
| **Platform Engineer** | IDP 자체 build/maintain |
| **SRE / Operations** | platform 운영 + 일반 service 운영 |
| **Security** | platform 의 security control 정의 |
| **Architect** | golden path 정의 |

→ Platform 은 "Application Developer" 의 product.

---

## 4. self-service의 spectrum

```
[chatops / ticket]           ↔   [GitOps]   ↔   [UI / CLI]    ↔   [IaC PR]

slow but controlled          ←─────────────────────→        fast but danger
```

→ 일반 변경 = self-service, 고위험 변경 = approval.

---

## 5. golden path (★)

```
한 줄로 새 service:
  $ idp scaffold create my-app --type spring-boot

자동 생성:
  ✓ git repo (template 적용)
  ✓ CI/CD pipeline
  ✓ dev/stage namespace
  ✓ Postgres / Redis (필요 시)
  ✓ secrets store entry
  ✓ monitoring dashboard
  ✓ alert rule
  ✓ on-call group
  ✓ IAM role
  ✓ Backstage entity

개발자 작업: code + Dockerfile (template 에 이미 있음)
```

---

## 6. T-shape engineer

```
폭 (다양 기술 얕게):   ████████████
깊이 (한 영역 깊게):       ██
                          ██
                          ██

→ platform 이 폭을 흡수 → 개발자가 깊이 집중.
```

---

## 7. product mindset (★ 핵심)

```
platform team = product team:
  - 사용자 (개발자) 조사 (interview / survey)
  - roadmap (quarterly)
  - metric (DX score, adoption)
  - 문서 / 학습 자료
  - sprint / release
  - feedback loop

❌ 안 좋은 platform:
  "여기 있어요. 알아서 쓰세요."

✅ 좋은 platform:
  "어떤 어려움이 있나요?" — onboarding session
  "이번 분기에 X feature 출시" — 공유
  "DX score 75 → 85" — 측정
```

---

## 8. metric (DX)

```
- Time to first deploy (new dev)        — 1주 → 1일
- Time to create new service             — 1일 → 30분
- Time to setup local env                — 4h → 30min
- 개발자 만족도 (NPS / survey)             — 30 → 70
- platform 사용률 (adoption)              — 50% → 95%
- ticket / question 빈도                  — 50/week → 5/week
```

---

## 9. anti-pattern

```
❌ platform 을 강제 — 매력 없으면 우회.
❌ platform 만 만들고 사용자 없음 — 광고 / training 필요.
❌ DevOps team 이름만 platform team — mindset 다름.
❌ 너무 많은 abstraction — leaky abstraction = 더 복잡.
❌ 단일 거대 platform — 모듈화.
```

---

## 10. 도입 단계

```
Stage 1: pain point 식별 (개발자 interview)
Stage 2: thinnest viable platform (MVP — golden path 1개)
Stage 3: pilot team (자발적 사용)
Stage 4: 확대 + feedback
Stage 5: 전사 표준
Stage 6: 성숙 (advanced feature)
```

→ **bottom-up 자연 채택 > top-down 강제**.

---

## 11. 함정

1. **너무 많은 customization 허용** — 다시 fragmentation.
2. **너무 적은 customization** — 사용자 우회.
3. **platform team 이 black box** — 외부 contribute 어려움.
4. **upgrade strategy 없음** — old version 누적.
5. **document 부재** — onboard 못 함.
6. **사용자 interview 안 함** — 사용자 욕구 모름.

---

## 12. 관련

- [[platform-engineering|↑ platform-engineering]]
- [[backstage]]
- [[scaffolding]]
- [[developer-experience]]
