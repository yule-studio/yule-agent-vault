---
title: "Capacity planning (SRE) — forecast / error budget / business ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T12:07:00+09:00
tags: [devops, sre, capacity-planning]
---

# Capacity planning (SRE) — forecast / error budget / business ★

**[[sre|↑ sre]]**

---

## 1. SRE 관점의 capacity

```
performance 관점:
  → "어디서 느린지" (지금)

SRE 관점:
  → "언제 깨질지" (미래)
  → SLO 안에서 buffer
  → business 와 align
  → cost vs reliability trade-off
```

→ [[../performance/capacity-planning|performance/capacity-planning]] 가 기술. 여기는 SRE process.

---

## 2. 핵심 질문 (★)

```
1. peak traffic 어디?
2. growth rate 의 prediction?
3. SLO 안 깨기 위한 headroom?
4. 한 component 가 fail 의 영향?
5. dependency 의 limit?
6. cost optimal point?
7. 이상한 event (Black Friday) 대응?
8. degraded mode 의 capacity?
```

---

## 3. 4 phase process

```
1. Measure (baseline)
   - 현재 traffic / resource / SLO

2. Forecast (예측)
   - business growth
   - seasonality
   - new feature

3. Plan (대응)
   - scale strategy
   - architectural change
   - cost approval

4. Validate (검증)
   - load test
   - chaos
   - canary
```

→ quarterly 반복.

---

## 4. Measure (baseline) (★)

```
metric (per service):
  - peak RPS (per day / week)
  - p50 / p95 / p99 latency
  - error rate
  - CPU avg / peak
  - memory avg / peak
  - DB connection / IOPS
  - cache hit rate
  - external API call rate / latency
  - request 의 cost ($)

수집 기간:
  최소 1 month (seasonal pattern)
  3 month (trend)
  1 year (annual)
```

---

## 5. SLO 와의 연결 (★)

```
SLO 가 capacity 의 ceiling:
  
  current p99 latency: 200ms
  SLO target: < 500ms
  → headroom 300ms (60%)

current error rate: 0.1%
SLO target: < 0.5%
→ budget 0.4% (80%)

→ 현재 SLO 안 → growth 가능
→ SLO 위반 임박 → 즉시 scale
```

→ "SLO compliance" = "capacity OK".

---

## 6. Error Budget Policy 와 (★)

```
budget 의 burn rate:
  
  hot path 에서:
    capacity 부족 → error rate ↑ → budget 빨리 소모
    → "이번 분기 budget 의 50%" 소모 시 → scale 결정
    
  burn rate alert:
    14.4x burn (1h window) → page
    3x burn (6h window) → ticket
```

→ capacity planning = budget 보호.

---

## 7. growth forecast

```
business input:
  - 마케팅 spend
  - DAU / MAU growth
  - 신기능 release
  - geographic expansion
  - 매출 target

기술적 변환:
  - DAU N → RPS = N × avg req/user/day / 86400
  - 새 feature 의 estimate
  - peak/avg ratio (보통 5-10x)

scenarios:
  - base case (예상)
  - growth case (낙관, 50%↑)
  - stress (200%↑, Black Friday)
```

---

## 8. forecast 방법

```
A. linear extrapolation
   "지난 N 달 평균 trend" 연장
   
B. seasonal
   "지난 해 12월 의 traffic × growth"
   
C. ML forecast
   Prophet (Facebook)
   ARIMA
   Holt-Winters
   
D. business-driven
   "이번 분기 N% growth target"
   marketing → engineering 변환

도구:
  - Prophet / statsmodels (Python)
  - PromQL predict_linear()
  - Datadog Forecast
  - 자체 BigQuery / Snowflake
```

```promql
# 1주 후 예측 (선형)
predict_linear(http_requests_total[7d], 7 * 24 * 3600)
```

---

## 9. headroom 정책 (★)

```
일반 service:
  CPU peak < 70%        → 30% headroom
  memory peak < 80%
  DB connection < 80%
  
critical service:
  peak < 50%            → 50% headroom (재해 대비)
  
batch / non-critical:
  peak ≤ 90% OK         → 10% headroom

→ headroom 적을수록 cost 적지만 risk ↑.
```

---

## 10. capacity calculation 예

```
현재:
  100,000 DAU
  peak RPS: 1,000
  10 server (each 100 RPS, CPU 50%)
  
forecast (3개월 후):
  200,000 DAU
  peak RPS: 2,000

계산:
  base case:
    same server, CPU 100% → 부족
    20 server 필요 (CPU 50%)
    
  margin:
    Black Friday 4x = 8,000 RPS
    → 80 server 또는 autoscale 의 max
    → DB / cache 의 scale 도
    
  cost:
    20 server × $50/mo = $1,000/mo
    또는 spot mix = $500/mo
```

---

## 11. dependency limit (★)

```
service 자체 scale OK 도:
  
  DB:
    max_connections 500
    IOPS / storage
    replica 의 read 한계
    
  Redis:
    memory limit
    network bandwidth
    
  Kafka:
    partition 수
    consumer scale
    
  external API:
    rate limit
    quota
    cost per call
    
→ chain 의 weakest link.
```

→ 같은 forecast 의 모든 dependency 까지 확인.

---

## 12. autoscaling 신뢰

```
HPA / Karpenter / KEDA 만 의존 X:
  
  cold start:
    JVM 30-60s
    Lambda warm pool 비용
    
  DB connection:
    autoscale 가 따라가도 connection 부족
    
  rate limit (외부):
    autoscale 도 외부 limit 못 넘김
    
  spike:
    분 단위 scale 가 분 미만 spike 못 막음

→ autoscale + buffer (pre-warm) + circuit break.
```

---

## 13. cost 결정

```
sizing 의 두 비용:
  
  under-provisioning:
    - SLO 위반
    - 매출 loss
    - 사용자 이탈
    - 평판 손해
    
  over-provisioning:
    - cloud cost ↑
    - utilization ↓
    
sweet spot:
  SLO 안 깨기 + 사용 80% (이전).
  
business 와 align:
  "$X 더 쓰면 SLO 99.9 → 99.95"
  ROI 계산.
```

---

## 14. capacity exercise (★ 분기)

```
시뮬레이션:
  1. "지금 traffic 2x 면?" 
     → 어느 component 깨지나?
     → scale plan?
     
  2. "지금 부터 1 month 정지 후 시작?"
     → cold start?
     → 사용자 영향?
     
  3. "Black Friday 5x"
     → pre-warm strategy?
     → degraded mode?
     
  4. "외부 service 30% slow"
     → cascade?
     → circuit break working?

도구:
  - load test (k6 / wrk)
  - chaos (Chaos Mesh)
  - game day
```

---

## 15. business align

```
quarterly review meeting:
  - 지난 분기 actual vs forecast
  - 다음 분기 forecast
  - 큰 launch / campaign
  - architectural debt
  - cost projection
  - SLO compliance
  - capacity gap
  
attendees:
  - Engineering leadership
  - SRE
  - Product
  - Finance (cost)
  - Marketing (campaign)
  
output:
  - action item
  - approval (큰 cost)
  - alert / pre-warm 계획
```

---

## 16. capacity model

```
간단 model:
  
  N (users) → R (requests) → C (capacity)
  
  R = N × avg_req_per_user
  C_required = R × peak_multiplier / capacity_per_server
  cost = C × server_cost
  
복잡 model:
  
  여러 service / DB / cache:
    각 component 의 capacity formula
    dependency graph
    spreadsheet 또는 Python notebook
```

---

## 17. 흔한 함정 (★)

```
1. forecast 의 single number
   → range (P50 / P95)

2. autoscale 만 의존
   → DB / 외부 service 까지 검토

3. seasonal 무시
   → 연 1회 vs 일별 vs 시간별

4. cost projection 없음
   → 갑자기 $$ 폭주

5. SLO 와 disconnect
   → capacity OK 인데 SLO 위반

6. degraded mode 없음
   → 100% 아니면 0%

7. dependency 의 limit 무시
   → DB 부터 깨짐

8. test 의 environment 다름
   → staging 작아서 의미 X

9. organizational misalignment
   → engineering 가 capacity 알아도 product 가 모름

10. review 안 함
    → quarterly 정기.
```

---

## 18. runbook

```
capacity 부족 발생 시:

immediate (5 min):
  ☐ autoscale 동작?
  ☐ rate limit 켜기 (lower priority)
  ☐ cache 적극
  ☐ circuit break 외부
  ☐ degraded feature off

short-term (1 hour):
  ☐ horizontal scale 수동
  ☐ vertical scale (DB)
  ☐ read replica
  ☐ traffic shift

medium-term (1 day):
  ☐ optimize query
  ☐ index 추가
  ☐ cache 정책 강화

long-term (sprint):
  ☐ architectural change
  ☐ sharding
  ☐ regional split
```

---

## 19. 관련

- [[sre|↑ sre]]
- [[slo-management]]
- [[../performance/capacity-planning|↗ performance capacity]]
- [[../finops/right-sizing|↗ FinOps]]
- [[load-shedding]]
