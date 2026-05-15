---
title: "SLO management — 운영"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T06:08:00+09:00
tags: [devops, sre, slo]
---

# SLO management — 운영

**[[sre|↑ sre]]**

---

## 1. SLI/SLO/Error Budget — 정의

→ [[../monitoring/slos-and-sli|monitoring/slos-and-sli]] 참조.

여기서는 **운영 측면** 다룸 — 누가 정하고, 누가 본다, 위반 시 어떻게.

---

## 2. SLO 정하는 흐름

```
1. user journey 식별 (login / search / checkout)
2. journey 별 핵심 SLI 선정 (latency, availability, ...)
3. 현재 SLI 측정 (1-2주 baseline)
4. SLO 제안 (baseline 보다 약간 보수적)
5. stakeholder 협의 (product / engineering / leadership)
6. dashboard + alert + 정책 문서화
7. 매 quarter 재검토
```

---

## 3. CUJ (Critical User Journey)

| journey | SLI | SLO |
| --- | --- | --- |
| login | success rate, p95 latency | 99.9%, < 500ms |
| search | p95 latency, freshness | < 1s, < 5min stale |
| checkout | success rate, p99 latency | 99.95%, < 2s |
| payment | success rate | 99.99% (★ business critical) |

---

## 4. Error Budget Policy (★)

```markdown
# Error Budget Policy — checkout

## SLO
- 99.95% success rate, 28d rolling window
- Budget: 0.05% = ~20 min downtime / month

## Burn 단계
| 소모 | 액션 |
| --- | --- |
| < 25% | normal — 전체 deploy OK |
| 25-50% | warning — risky change 는 staging 더 길게 |
| 50-75% | feature 배포 동결, 리스크 감수 작업 금지 |
| 75-100% | 즉시 reliability 작업만, 사후 retrospective |
| > 100% | dev team 전원 reliability 만, leadership 보고 |

## 위반 시
- product manager 보고 (24h)
- engineering lead 가 reliability 작업 우선순위
- 다음 sprint planning 에서 capacity 의 30% 이상을 reliability
```

---

## 5. burn rate alert — 권장 multi-window

| Severity | short window | long window | burn rate | 의미 |
| --- | --- | --- | --- | --- |
| page | 5m | 1h | 14.4x | 1h 안에 2% budget 소모 |
| page | 30m | 6h | 6x | 6h 안에 5% 소모 |
| ticket | 2h | 1d | 3x | 1d 안에 10% 소모 |
| ticket | 6h | 3d | 1x | 3d 안에 10% 소모 |

→ short + long 둘 다 충족 시만 fire (false positive 감소).

---

## 6. SLO dashboard 표준 (Grafana)

```
Row 1: SLI 현재 (28d rolling) — 큰 숫자
Row 2: SLO 목표 + Error Budget 남은 % (gauge)
Row 3: Burn rate (1h / 6h / 1d / 3d)
Row 4: SLI trend (28d time series + SLO threshold line)
Row 5: budget 소모 시계열 (어디서 빠졌나)
Row 6: recent incident annotation
```

---

## 7. SLO 재조정

```
quarterly review:
- 지난 quarter SLO compliance?
- 사용자 불만 (NPS, support ticket) 과 일치?
- 너무 자주 위반? → 더 느슨 OR 코드 개선
- 거의 안 위반? → 더 엄격 OR 다른 SLI 선택
```

→ SLO 는 **계약** 이 아니라 **합의** — 정기 재조정.

---

## 8. 함정

1. **SLO 가 product / leadership 과 합의 안 됨** — 의미 없음.
2. **internal latency 만** — frontend / CDN 까지 포함 안 함.
3. **business critical 과 정렬 안 됨** — 매출 영향 무시.
4. **error budget freeze 안 지킴** — 정책 의미 없음.
5. **SLO 가 너무 많음** — 우선순위 모호.

---

## 9. 관련

- [[sre|↑ sre]]
- [[../monitoring/slos-and-sli]]
- [[../monitoring/alerting]]
- [[postmortem]]
