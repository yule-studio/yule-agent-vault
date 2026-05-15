---
title: "SLI / SLO / SLA / Error Budget"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:48:00+09:00
tags: [devops, monitoring, slo, sli, sre]
---

# SLI / SLO / SLA / Error Budget

**[[monitoring|↑ monitoring]]**

---

## 1. 용어

| | 무엇 | 예 |
| --- | --- | --- |
| **SLI** (Indicator) | 측정 | latency p95, error rate, availability |
| **SLO** (Objective) | 목표 | 99.9% requests < 500ms |
| **SLA** (Agreement) | 계약 (외부) | violation 시 환불 / 페널티 |
| **Error Budget** | 1 - SLO | 0.1% (월 43분) 까지 실패 허용 |

---

## 2. 왜 필요

### 왜 필요한가
- "가능한 빨리" 는 측정 불가 → 객관 기준 필요.
- SLO 가 deploy 속도 / risk taking 의 기준.
- Error Budget = product velocity 와 reliability 의 균형.

### 안 하면 문제
- 우선순위 모호 — "느린 것 같은데?" "괜찮은 것 같은데?"
- 무한 reliability 추구 → 기능 stagnation.
- SLA 위반 모르고 가다 환불.

### 대안
- 100% availability 목표 — 비용 무한.
- "best effort" 명시 안 함 — 모호.

### 트레이드오프
- SLO 너무 엄격 → deploy 정지 빈번.
- SLO 너무 느슨 → 사용자 불만 모름.

---

## 3. SLI 종류

| Category | 예 |
| --- | --- |
| **Availability** | success rate (2xx + 3xx + 4xx) / total |
| **Latency** | p50 / p95 / p99 |
| **Throughput** | requests per second |
| **Correctness** | output correct (정답 비율) |
| **Quality** | resolution / bit rate (영상) |
| **Freshness** | data 최신성 (last update) |

---

## 4. SLO 정의 (Google SRE)

```
For 99.9% of requests in any rolling 28-day window,
the latency measured at the load balancer
must be less than 500ms.
```

→ "99.9%" (목표), "28-day rolling" (window), "load balancer" (측정 위치), "500ms" (임계).

---

## 5. Error Budget 계산

| SLO | 월 허용 downtime |
| --- | --- |
| 99% | 7시간 14분 |
| 99.5% | 3시간 38분 |
| 99.9% (three nines) | 43분 |
| 99.95% | 21분 |
| 99.99% (four nines) | 4분 26초 |
| 99.999% (five nines) | 26초 |

→ **99.9% 가 대부분의 SaaS 표준**. five nines = 매우 비쌈.

---

## 6. PromQL — SLI 측정

```promql
# Availability SLI (지난 28일)
sum(rate(http_requests_total{status!~"5.."}[28d]))
/ sum(rate(http_requests_total[28d]))

# Latency SLI (p95 < 500ms 비율)
sum(rate(http_duration_bucket{le="0.5"}[28d]))
/ sum(rate(http_duration_count[28d]))

# Error Budget remaining
1 - (
    (1 - <availability>) / (1 - 0.999)
)
```

---

## 7. Burn Rate Alert (★ Google SRE)

```
Burn Rate = (실제 error rate) / (허용 error rate)

Burn Rate = 1.0 → 정확히 SLO 만큼 소모
Burn Rate = 10.0 → 10배 빠르게 budget 소모 → 알람
```

```yaml
# 짧은 window + 긴 window 동시 평가
- alert: ErrorBudgetBurnFast
  expr: |
    (rate(errors[5m]) / rate(total[5m])) > (14.4 * 0.001)  # 1% in 1h
    and
    (rate(errors[1h]) / rate(total[1h])) > (14.4 * 0.001)
  for: 2m
  labels: {severity: page}

- alert: ErrorBudgetBurnSlow
  expr: |
    (rate(errors[1h]) / rate(total[1h])) > (3 * 0.001)
    and
    (rate(errors[6h]) / rate(total[6h])) > (3 * 0.001)
  for: 15m
  labels: {severity: ticket}
```

---

## 8. Error Budget Policy

> "Error Budget 이 50% 소모되면, 새 기능 배포 동결. 100% 소모되면, 신뢰성 작업만."

→ product 팀 vs SRE 갈등의 해결책.  
→ "더 배포" vs "더 안정" 의 정량적 기준.

---

## 9. SLO 대시보드 (Grafana)

```
panel 1: 현재 SLI (28일 rolling)        | 99.94%
panel 2: SLO 목표                       | 99.9%
panel 3: Error Budget 남은              | 64%
panel 4: Burn rate (현재)                | 0.4x
panel 5: 시간별 SLI trend
```

---

## 10. 함정

1. **너무 많은 SLO** — 우선순위 모호. 서비스당 3-5개로.
2. **사용자 관점 아님** — internal latency 만 측정 (frontend 까지 안 봄).
3. **availability 100% 목표** — 신뢰 stagnation.
4. **window 너무 짧음** (1일) — flapping.
5. **error budget freeze 안 지킴** — 정책 의미 없음.

---

## 11. 관련

- [[monitoring|↑ monitoring]]
- [[alerting]]
- [[prometheus]]
- [[../sre/sre|↗ sre]]
