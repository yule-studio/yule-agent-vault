---
title: "Cardinality management — Prometheus / observability cost ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:35:00+09:00
tags: [devops, monitoring, cardinality, prometheus]
---

# Cardinality management — Prometheus / observability cost ★

**[[monitoring|↑ monitoring]]**

---

## 1. 무엇

```
cardinality = 한 metric 의 unique time series 개수.

예:
  http_requests_total{method, status, path}
  
  method: 5 (GET, POST, PUT, DELETE, PATCH)
  status: 6 (200, 201, 400, 401, 404, 500)
  path:   100
  
  → 5 × 6 × 100 = 3,000 series

cardinality 폭증 = Prometheus 의 OOM / cost 폭주.
```

---

## 2. 왜 시니어가 알아야

```
회사 성장:
  - 1 service → 100 service
  - 100 endpoint → 10,000
  - 10 status → 모든 customer ID

→ 백만 ~ 천만 series 폭주
→ Prometheus 1 → memory 100GB+
→ 비용 폭주 (Datadog $$$/host)

매년 monitoring cost 가 가장 빨리 증가하는 영역.
```

---

## 3. 폭증 원인 (★)

```
높은 cardinality label:
  - user_id      (100만+ 사용자)
  - request_id   (개별 request)
  - email
  - session_id
  - trace_id
  - full URL path (with query)
  - IP address
  - timestamp

→ 절대 label 으로 X.
```

---

## 4. 안전한 label

```
유한 / 작은 set:
  - environment (dev/staging/prod)
  - cluster (3-10 cluster)
  - region (3-5 region)
  - service (수십)
  - method (HTTP method 5-10)
  - status (HTTP status 카테고리)
  - endpoint pattern (/api/users/:id, not /api/users/123)
```

→ template 화. `/users/123` X, `/users/:id` O.

---

## 5. 측정

```promql
# 가장 많은 series 의 metric
topk(20, count by (__name__)({__name__=~".+"}))

# 한 metric 의 series 수
count(http_requests_total)

# label 별 cardinality
count by (path) (http_requests_total)

# Prometheus 자체 stat
prometheus_tsdb_head_series
scrape_samples_post_metric_relabeling
```

```bash
# Prometheus / cli
curl -s http://prom:9090/api/v1/status/tsdb | jq '.data.seriesCountByMetricName' | head -20
```

---

## 6. 대응 (★)

### A. label 제거 (drop)

```yaml
# scrape config
metric_relabel_configs:
  - source_labels: [__name__]
    regex: 'http_requests_total'
    target_label: 'request_id'
    replacement: ''
    action: replace
  
  # 또는 drop 자체
  - source_labels: [__name__, request_id]
    regex: 'http_requests_total;.*'
    action: drop
```

→ Prometheus 측에서 거름.

### B. label aggregation

```yaml
# bucket / group 으로 reduce
metric_relabel_configs:
  - source_labels: [status_code]
    regex: '(2..|3..|4..|5..)'
    target_label: status_class    # 2xx, 3xx, 4xx, 5xx 만
  - source_labels: [status_code]
    action: drop
```

### C. recording rule (★ aggregate 미리)

```yaml
# recording rule
- record: http:requests_rate5m:by_status
  expr: |
    sum by (service, status_class) (
      rate(http_requests_total[5m])
    )
```

→ 큰 query 의 결과를 저장. dashboard 빠름.

### D. histogram bucket 수 조정

```yaml
# Spring Micrometer
management:
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true
      sla:
        http.server.requests: 100ms,500ms,1s    # 3 bucket 만
```

→ default 가 14 bucket → 3 으로 줄임 = cardinality 4-5x ↓.

### E. tag 절제

```kotlin
// application 코드에서
@Timed(value = "api.duration", extraTags = ["endpoint", endpoint])
public Response handle() { ... }

// extraTags 의 endpoint = pattern (/users/:id)
//   ❌ /users/123 (수백만)
//   ✓ /users/:id (몇 십)
```

---

## 7. exemplar (★ trace 와 연결)

```
metric cardinality 부담 없이 자세한 trace 가능:
  - metric: low cardinality (HTTP method, status)
  - exemplar: trace_id 포함 (개별 request)
  - Grafana 가 metric → trace 점프

→ "느린 request 분석" 시:
   metric → exemplar (trace_id) → Tempo trace
```

```yaml
# Spring Boot
management:
  tracing:
    sampling:
      probability: 0.1
  observations:
    http:
      server:
        requests:
          name: http.server.requests
```

---

## 8. 큰 organization 의 도구

```
single Prometheus:
  10M series 까지

scale-out:
  Thanos (★ OSS)
    - 여러 Prometheus 합침
    - long-term S3 storage
    - global query
  
  Cortex / Mimir (★ Grafana)
    - multi-tenant
    - 수억 series
  
  VictoriaMetrics
    - 단일 binary, 빠름
    - cardinality 친화
```

→ 큰 회사 = Mimir / Thanos / VictoriaMetrics. cost 큼.

---

## 9. cost 비교 (★)

```
self-host Prometheus + Thanos:
  - 5 server × 8GB = $400/mo
  - S3 = $100/mo
  - 운영 부담 (사람 시간)
  - 큰 회사면 10M+ series 가능

Datadog:
  - $15/host/mo
  - + $0.10/metric/mo (custom)
  - + $0.50/GB log
  - 100 host = $1500 base
  - + custom metric 폭주 시 $10k/mo

Grafana Cloud:
  - free tier 10k series
  - $0.0008/series/mo (Pro)
  - 10M series = $8k/mo

→ cardinality 가 cost 의 가장 큰 lever.
```

---

## 10. anti-pattern

```
❌ "혹시 모르니 다 label"
   → 모든 metric 에 user_id, request_id 추가
   → cardinality 폭주

❌ application 코드에서 metric name 동적
   metricRegistry.counter("api_" + endpoint + "_total")
   → 수천 metric name

❌ histogram bucket 100개
   → bucket × series 폭주

❌ infinite retention
   → S3 archive 로 옮김

❌ 모두 5s scrape
   → 1m / 5m 도 OK
```

---

## 11. 실전 점검 (★ quarterly)

```
1. topk 의 cardinality metric
2. 의도 없는 high cardinality 식별
3. 폐기 candidate (안 쓰는 metric)
4. recording rule 로 pre-aggregate
5. histogram bucket review
6. retention 정책
7. cost vs value 평가
```

```bash
# 안 쓰는 metric 찾기 (Grafana / dashboard 에서 안 reference)
# https://github.com/grafana/mimirtool 의 unused-metrics
mimirtool analyze prometheus --address http://prom:9090 \
    --output prom-metrics.json

mimirtool analyze grafana --address http://grafana:3000 \
    --output grafana-used.json

mimirtool analyze ruler --rule-dir ./rules \
    --output rules-used.json

# unused = prom 의 metric - grafana 의 used - rules 의 used
```

---

## 12. log 의 cardinality

```
같은 문제 log 에도:
  - JSON 의 dynamic field
  - high cardinality structured log
  - log aggregation tool (Loki / Elastic) 의 비용
  
대응:
  - log 의 field 표준화
  - PII 제거
  - 자주 query 하지 않는 정보는 trace 로
```

---

## 13. trace 의 cardinality

```
distributed trace:
  - 매 request 의 unique trace_id
  - span attribute (user_id, full URL)
  
대응:
  - sampling rate (1% 등)
  - tail-based sampling (error 만 100%)
  - span attribute 절제
```

---

## 14. 함정

1. **user_id label** — 가장 흔한 폭증.
2. **dynamic metric name** — application 코드의 string concat.
3. **histogram bucket 너무 많음** — bucket × series.
4. **retention 무한** — S3 / Glacier.
5. **anomaly metric 만들 때마다 새 metric** — 기존 metric 의 label.
6. **상용 도구의 ingestion cost 무지** — Datadog custom metric 비쌈.
7. **monitoring 자체 의 monitoring 없음** — Prometheus OOM.

---

## 15. 관련

- [[monitoring|↑ monitoring]]
- [[prometheus]]
- [[application-metrics]]
- [[../finops/architecture-optimization|↗ cost optimization]]
