---
title: "On-call rotation"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T06:13:00+09:00
tags: [devops, sre, on-call]
---

# On-call rotation

**[[sre|↑ sre]]**

---

## 1. on-call 이란

- 정해진 시간에 page (호출) 받을 책임자.
- alert 발생 시 즉시 응답 (10-15분 SLA).
- 1차 (primary) + 2차 (secondary) + escalation manager.

---

## 2. 왜 필요

### 왜 필요
- 24/7 서비스는 사람이 24/7 있어야.
- 명확한 책임자 없으면 "누가 봐?" 무한 대기.

### 안 하면 문제
- alert 무시 → SLA 위반 → 매출 손실.
- 특정 사람만 대응 → burnout / bus factor 1.

### 대안
- 모두가 항상 응답 — 비현실적.
- 외주 NOC — 도메인 지식 부족.

### 트레이드오프
- on-call = 부담스러움 (보상 / 시간 보전 필요).
- rotation 너무 빠름 = 컨텍스트 단절.

---

## 3. rotation 패턴

### 3-1. weekly rotation (가장 흔함)

```
| 주 | primary    | secondary |
| 1  | alice      | bob       |
| 2  | bob        | carol     |
| 3  | carol      | dave      |
| 4  | dave       | alice     |
```

→ 1주 단위 (월 → 일 21:00 등으로 handoff).

### 3-2. follow-the-sun (글로벌)

```
| 시간대 | 담당 region |
| 00-08 UTC | APAC |
| 08-16 UTC | EMEA |
| 16-24 UTC | Americas |
```

→ 야간 page 없음, region 마다 day-time 만.

### 3-3. primary + secondary + manager

```
Primary    — 즉시 응답 (10min)
Secondary  — primary 응답 X 시 (30min)
Manager    — secondary 도 X 시 / P0 (1h)
```

---

## 4. PagerDuty Schedule 예

```
Service: web-api
Escalation Policy: web-api-ep
  Level 1: Schedule "web-api-primary" (week rotation) → SMS + call
  Level 2: Schedule "web-api-secondary" → SMS
  Level 3: User "eng-lead" → call
  Repeat: 2 times

Integration: AlertManager → PagerDuty events API
```

---

## 5. handoff 미팅 (★)

매주 (또는 rotation 끝) 30분:

```markdown
## Handoff — week N

### 진행 중 incident
- INC-001: checkout 500 errors → 어제 임시 패치, root cause 미정
- INC-002: search latency spike → fix PR #234 merged, 모니터 중

### 모니터 필요
- DB master CPU 70% — 다음 주 scale up 예정
- Kafka consumer lag — auto-scale 잘 동작?

### known issue (page 받지 말 것)
- alert "DiskUsageHigh" — 이미 PR 진행 중 (cleanup job)

### unfinished runbook
- DB failover 절차 — 검증 미완

### 인수자 질문 환영
```

---

## 6. on-call shift 표준

| 시간대 | 응답 SLA |
| --- | --- |
| 업무시간 (09-18) | 5min |
| 야간 (18-09) | 15min |
| P0 (서비스 down) | 즉시 (있는 곳에서) |
| P1 (degraded) | 15min |
| P2 (warning) | 1h |

---

## 7. on-call 보상

- 추가 수당 (예: weekly $500 + page 당 $50)
- comp time (page 후 다음 날 휴식)
- 인사고과 반영
- 회사 정책에 따라 매우 다름.

---

## 8. burnout 방지

```
- shift 7일 초과 X (피로 누적)
- 페이지 후 충분한 휴식 (8h 권장)
- rotation 4명 이상 (1명 빠져도 OK)
- toil 줄이기 (★ — page 자체를 줄임)
- 야간 page 5건/주 초과 시 → root cause 분석
```

→ **on-call 의 진짜 KPI = "page 가 거의 안 와도 시스템이 잘 돈다"**.

---

## 9. 함정

1. **bus factor 1** — 1명만 on-call 가능 → 휴가 / 퇴사 시 disaster.
2. **noisy alert** — 진짜 page 와 noise 구분 불가 → fatigue.
3. **runbook 없음** — 새벽 3시에 무엇을 해야 할지 모름.
4. **handoff 없음** — 이전 issue 모르고 시작.
5. **escalation 없음** — primary 자고 있으면 무한 대기.
6. **보상 없음** — on-call 기피 / 인재 이탈.

---

## 10. 관련

- [[sre|↑ sre]]
- [[incident-response]]
- [[runbook]]
- [[../monitoring/alerting]]
