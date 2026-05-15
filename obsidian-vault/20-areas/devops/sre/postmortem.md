---
title: "Blameless Postmortem"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T06:19:00+09:00
tags: [devops, sre, postmortem]
---

# Blameless Postmortem

**[[sre|↑ sre]]**

---

## 1. postmortem 이란

incident 가 끝난 후, **체계적으로 학습** 하기 위한 회고.

- "왜 발생했나" + "다시 안 일어나려면"
- **blameless** = 사람을 비난하지 않음, system 개선.
- 모두 공유 (전사 / public). 학습 = 자산.

---

## 2. 왜 필요

### 왜 필요
- 같은 incident 반복 = 비용.
- 암묵지 → 명시지 (다른 사람도 학습).
- 신뢰 (transparency).

### 안 하면 문제
- 같은 실수 매주.
- 사람 비난 분위기 → 사실 숨김 → 학습 0.
- 회사 차원 reliability 정체.

### 대안
- "그냥 알아서 잘 하자" — 시스템적 개선 X.
- 비공개 postmortem — 다른 팀이 같은 실수.

### 트레이드오프
- 시간 소요 (incident 당 4-8h).
- 자세하면 공개 부담 (외부 PR 영향).

---

## 3. 표준 템플릿

```markdown
# Postmortem — INC-2026-0512-001

## Summary (TL;DR)
- 1-2줄. 무엇이 일어났고, 영향은.

## Status
- Author: alice
- Status: complete (draft / in-review / complete)
- Date: 2026-05-12

## Impact
- 사용자: ~12,000명 영향
- 시간: 13:42 ~ 14:10 UTC (28분)
- 매출: ~$8,400 손실
- SLO: checkout SLO 28일 budget 의 35% 소모

## Timeline (UTC)
| 시간 | 이벤트 |
| --- | --- |
| 13:30 | deploy v1.5.2 to prod |
| 13:42 | error rate spike, AlertManager fired |
| 13:43 | on-call (Bob) ack |
| 13:48 | rollback initiated |
| 13:51 | rollback complete |
| 13:55 | error rate back to baseline |
| 14:10 | status page green |

## Root Cause
- v1.5.2 의 OrderService.calculateTax() 가 N+1 query 도입
- 정상 traffic 에서는 250ms → checkout 시 DB pool exhaustion → 500 errors

## Detection
- AlertManager (HighErrorRate, > 5% for 2m) — 적절히 감지
- DB pool saturation alert 가 없었음 (개선 필요)

## Resolution
- v1.5.1 rollback (kubectl rollout undo)

## What went well ✅
- AlertManager 즉시 fire (2분 내)
- on-call 즉시 ack
- rollback 절차 명확 (3분 안에 complete)

## What went wrong ❌
- pre-prod 에서 load test 안 함 (small dataset 에서만 테스트)
- code review 시 N+1 못 잡음
- DB pool saturation alert 부재
- status page 업데이트 늦음 (10분 지연)

## Lucky factors 🍀
- 점심 시간 traffic 이라 영향 적었음 (peak 였으면 4배)
- on-call 이 회사에서 근무 중이었음 (집이었으면 5분 더)

## Action items
| ID | 액션 | 담당 | 마감 | 우선순위 |
| --- | --- | --- | --- | --- |
| A1 | OrderService N+1 fix (JOIN FETCH) | Bob | 2026-05-13 | P0 |
| A2 | CI 에 load test 단계 추가 | Carol | 2026-05-20 | P1 |
| A3 | DB pool saturation alert 추가 | Alice | 2026-05-17 | P1 |
| A4 | status page 자동 업데이트 (AlertManager → Statuspage) | Dave | 2026-05-30 | P2 |
| A5 | N+1 detection — code review 체크리스트 | Eve | 2026-05-25 | P2 |

## Lessons learned
- load test 는 prod-like dataset 필요.
- alert coverage 는 SLI 만이 아니라 resource saturation 포함.
- pre-prod canary 5min → 30min 으로 늘리자.
```

---

## 4. blameless 원칙

```
❌ "Bob 이 N+1 query 작성"
✅ "review process 가 N+1 을 잡지 못함"

❌ "Carol 이 status page 늦게 업데이트"
✅ "status page 업데이트가 manual — 자동화 부재"

❌ "Dave 가 alert 설정 안 함"
✅ "alert coverage 가 SLI 중심으로만 — resource 포함 안 됨"
```

→ **system / process 개선 으로 reframe**.

---

## 5. timeline 작성 팁

```
- UTC 표시 (전역 협업)
- 모든 시각은 자료에서 확인 (Slack / log / dashboard)
- "약 13:42" X — 정확한 시각
- 의사결정 시각 표시 ("13:48 IC: rollback 결정")
- impact 시작 / 종료 명시
```

---

## 6. action item — SMART

| 항목 | 좋은 예 | 나쁜 예 |
| --- | --- | --- |
| Specific | "DB pool saturation alert 추가 (HikariCP `connections_max - connections_active < 5`)" | "monitoring 강화" |
| Measurable | "load test threshold p99 < 500ms" | "성능 검증" |
| Assigned | "@alice" | "팀" |
| Realistic | "1주 안에" | "ASAP" |
| Time-bound | "2026-05-30" | "다음에" |

→ tracking — issue tracker 에 link.

---

## 7. retrospective vs postmortem

| | retrospective | postmortem |
| --- | --- | --- |
| 시점 | sprint / quarter | incident 후 |
| 범위 | process | 특정 사건 |
| 빈도 | 정기 | 발생 시 |
| 출력 | process 개선 | action item 풀 |

---

## 8. 함정

1. **blame** — "누가 했어" → 사실 숨김.
2. **action item 추적 X** — 작성하고 안 함 → 같은 incident 반복.
3. **template 만 채움** — root cause 안 깊이 파헤침 (5 Whys).
4. **공개 X** — 다른 팀 학습 0.
5. **timeline 부정확** — "약 ~ 분 후" → 검증 불가.
6. **lucky factor 빠짐** — 같은 incident 가 다른 조건에서 발생 시 더 심각.

---

## 9. 5 Whys 기법

```
Q1: 왜 error rate spike?
A1: DB pool 고갈

Q2: 왜 DB pool 고갈?
A2: N+1 query 가 connection 점유

Q3: 왜 N+1 query?
A3: tax 계산 시 각 line item 마다 별도 query

Q4: 왜 review 에서 못 잡았나?
A4: code review 가 logic 만 봄, query 안 봄

Q5: 왜 query 안 봄?
A5: review 체크리스트 없음

→ root cause: review process 의 query awareness 부재
→ action: review checklist + ORM N+1 detection 자동화
```

---

## 10. 관련

- [[sre|↑ sre]]
- [[incident-response]]
- [[runbook]]
