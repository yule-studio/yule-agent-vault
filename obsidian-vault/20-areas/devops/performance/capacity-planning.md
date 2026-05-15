---
title: "Capacity planning — load forecast / 예측"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:29:00+09:00
tags: [devops, performance, capacity]
---

# Capacity planning — load forecast / 예측

**[[performance|↑ performance]]**

---

## 1. 왜

```
"트래픽 X% 증가 시 무엇 깨지나?"
"3개월 후 회원 수 5배 되면 인프라 충분?"
"Black Friday 10x 대비 어떻게?"

→ 시니어 = 미래 예측 + 대비.
```

---

## 2. 4 단계

```
1. measure (현재 baseline)
2. forecast (미래 예측)
3. test (가설 검증)
4. provision (자원 확보)

→ 정기 반복 (quarter).
```

---

## 3. baseline 측정

```
현재 metric:
  - peak RPS / RPM
  - p50 / p95 / p99 latency
  - CPU avg / peak
  - memory avg / peak
  - DB connection / IOPS
  - cache hit rate
  - 외부 API call rate / latency
  - cost (per metric)

→ 1 month minimum. seasonality 보임.
```

---

## 4. growth 예측

```
business 의 input:
  - 회원 증가 (월 N%)
  - 매출 증가
  - 신기능 launch
  - 마케팅 campaign
  - 시장 확대 (geographic)

→ business 와 합의된 forecast.
```

---

## 5. headroom (★)

```
current peak CPU: 60%
headroom 정책:
  - 일반 50% 이내
  - peak 70% 이내
  - 사고 대비 80% 한계
  
→ 70% 넘으면 scale 결정.
   80% 가까이 = alert.
```

---

## 6. capacity 계산 예

```
현재:
  - 1000 RPS peak
  - 10 server × 100 RPS / server
  - CPU avg 50% / peak 70%

가설:
  - 3개월 후 2x traffic = 2000 RPS
  
계산:
  - 같은 server type → 20 server 필요
  - 또는 instance type ↑
  - DB: 2x query → connection pool / replica?
  - cache: 2x miss → Redis scale?
  - 외부 API: 2x call → rate limit?
  - network egress: 2x bandwidth
  - log volume: 2x → cost ↑
```

---

## 7. load test 와 결합

```
1. 1.5x load test
   → 어디 가장 먼저 깨지나?
   → CPU? DB? cache?

2. 2x load test
   → 추가 fix 영역?

3. 3x load test (★)
   → spike / Black Friday 대비
   → architectural change 필요?
```

→ test → fix → test 반복.

---

## 8. autoscaling 신뢰성

```
HPA / Karpenter 는 만능 X:
  - cold start (JVM 30s+)
  - DB connection 추가 limit
  - 외부 service rate limit
  - cache miss spike
  - LB / DNS TTL

→ "autoscale" 가정 X. 검증 필요.
```

---

## 9. peak 대비 (★)

```
black friday / new feature / TV 광고:

전략:
  1. pre-warm:
     - peak 1시간 전 N pod 미리
     - cache 미리 채움 (warm-up)
     
  2. circuit break:
     - 외부 service fail 시 degraded mode
     
  3. queue:
     - synchronous → async (Kafka)
     - degraded UX > 완전 fail
     
  4. read replica scale-out
  
  5. CDN 더 적극
  
  6. graceful degradation:
     - 추천 / personalization 끔
     - core 만 유지
     
  7. on-call standby
```

---

## 10. component 별 limit

```
PostgreSQL:
  - max_connections 200-500
  - 더 늘리면 PgBouncer 사용
  - shared_buffers RAM 의 25%
  
Redis:
  - clustered = 1000 keys/s 의 shard
  - hot key 의 한 shard 100k+
  
Kafka:
  - partition 의 throughput
  - consumer lag
  
LB:
  - ALB LCU
  - NLB connection
  
Network:
  - egress bandwidth
  - cross-AZ
```

→ 각 component 의 hard limit 인지.

---

## 11. cost projection

```
1.5x traffic:
  - compute: 1.5x (대략)
  - DB: 1.5-2x (read replica 추가 가능)
  - cache: 1.2x (hit rate 도 영향)
  - storage: linear
  - egress: 1.5x

총: 1.5 - 2x cost

→ business 의 ROI 계산 (매출 vs 비용)
```

---

## 12. seasonal pattern

```
e-commerce:
  - 매일 peak 19-22시
  - 주말 1.5x
  - 월말 sale 2-3x
  - Black Friday 5-10x
  - 연말 / 명절

학습:
  - 작년 metric 분석
  - YoY 비교
  - growth rate 적용
```

---

## 13. tool

| | 무엇 |
| --- | --- |
| **Prometheus + Grafana** | trend 시각화 |
| **AWS Compute Optimizer** | right-sizing 권장 |
| **Datadog** | predictive forecast |
| **자체 ML model** | seasonal + business input |
| **Excel / SQL** | 단순 |

```promql
# 7일 평균 RPS
avg_over_time(sum(rate(http_total[5m]))[7d:5m])

# 1주 후 예측 (선형)
predict_linear(http_total[7d], 7 * 24 * 3600)
```

---

## 14. capacity review meeting (★)

```
quarterly:
  - 지난 분기 actual vs predicted
  - 다음 분기 forecast
  - 큰 launch / campaign
  - architectural debt 점검
  - cost review
  - action item

attendees:
  - eng leadership
  - SRE
  - product
  - finance (cost)
```

---

## 15. 함정

1. **autoscale 만 의존** — DB / 외부 service 못 따라옴.
2. **linear 가정** — 실제는 non-linear (DB lock 등).
3. **best-case 만** — peak 시나리오 검토.
4. **forecast 단일 number** — range 사용 (P50 / P95).
5. **load test 가 prod 와 다른 size** — 결과 의미 X.
6. **dependency 의 limit 무시** — 외부 API rate.
7. **review 없이 운영** — 점진 깨짐.

---

## 16. 관련

- [[performance|↑ performance]]
- [[load-testing]]
- [[../sre/slo-management|↗ SLO]]
- [[../finops/right-sizing|↗ FinOps]]
