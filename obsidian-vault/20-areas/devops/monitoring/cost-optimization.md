---
title: "Observability cost optimization — log / metric / trace 종합 ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T12:05:00+09:00
tags: [devops, monitoring, cost-optimization, observability]
---

# Observability cost optimization — log / metric / trace 종합 ★

**[[monitoring|↑ monitoring]]**

---

## 1. 왜 비싸지나

```
observability 가 매년 가장 빨리 증가하는 cost:
  
  service 수 ↑    → metric / log 폭증
  micro-service  → service 마다 monitoring
  cloud-native    → ephemeral pod = 더 많은 metric
  trace          → request 당 detail
  high cardinality → label 폭주
  long retention  → 비싸진 storage

회사 monitoring cost 흔히 cloud cost 의 20-30%.
```

→ FinOps 의 대상.

---

## 2. cost 분해 (★)

```
observability 비용:

1. ingest (수집)
   - 50% 차지하는 부분 흔함
   - bytes/s × 시간

2. storage (저장)
   - retention 정책
   - hot vs cold

3. query (조회)
   - 분석 / dashboard load
   - 일부 도구는 별도 과금

4. agent / sidecar (compute)
   - 모든 pod 에 sidecar
   - memory / CPU 부담

5. egress (cross-region)
   - 본사 / 다른 region 으로
```

---

## 3. metric cost

```
Prometheus / Datadog:
  $ = series 수 × time

비용 결정:
  - cardinality (★ 가장 큰)
  - scrape interval (5s vs 15s vs 60s)
  - retention (15d vs 90d vs 1y)
  - storage type (local disk vs S3)
```

대응:
```
1. cardinality 축소
   → [[cardinality-management]] 참조
   
2. scrape interval ↑
   default 15s → 일부 30-60s
   (long-term aggregation 가 자세하면)
   
3. recording rule
   자주 query 의 pre-aggregate
   
4. drop / relabel
   안 쓰는 metric / label 제거
   
5. tier storage
   Prometheus (local 14d) + Mimir/Thanos (S3 long-term)
```

---

## 4. log cost (★ 가장 빨리 증가)

```
Datadog Logs:
  ingest:    $0.50/GB
  storage:   $1.27/GB-mo (15 days)
  
CloudWatch Logs:
  ingest:    $0.50/GB
  storage:   $0.03/GB-mo
  query:     $0.005/GB scanned
  
Loki:
  cheap (S3 storage)
  단 운영 부담

ELK Cloud:
  $/GB ingest + node $

1 TB / mo 의 회사:
  Datadog:    $1,770/mo (ingest+storage)
  CloudWatch: $510/mo (단 query 비쌈)
  Loki:       ~$50/mo (S3 + 약간 compute)
```

---

## 5. log 줄이기 (★)

### A. log level tuning

```yaml
# application.yml
logging:
  level:
    root: WARN          # ★ INFO 대신
    com.myorg: INFO
    org.springframework.web: WARN
    org.hibernate.SQL: WARN  # DEBUG → WARN
```

→ INFO 가 80%+. 대부분 noise. WARN+ 으로.

### B. 구조화 + 의미 있는 field

```java
// ❌ unstructured
log.info("user " + userId + " did " + action + " on " + resource);

// ✅ structured
log.info("user_action", 
    kv("user_id", userId),
    kv("action", action),
    kv("resource", resource));
```

→ Loki / ES 에서 query 효율. 같은 message 의 deduplication.

### C. sampling

```java
// 정상 200 OK 의 1% 만 log
if (response.status == 200 && random() > 0.01) {
    return;  // skip
}
log.info(...);
```

→ error 100% / success 1% pattern.

### D. drop noise

```yaml
# Fluent Bit / Vector filter
[FILTER]
    Name        grep
    Match       app.*
    Exclude     log /healthz
    
[FILTER]
    Name        modify
    Match       *
    Remove      thread_name
    Remove      class_name_fqn
```

→ healthcheck log / verbose field 제거.

### E. retention 차등

```
hot (검색 자주):    7 day
warm:               30 day
cold (audit 만):    1 year (S3 Glacier)
```

→ 30일 후 S3 archive. ELK 의 ILM / Loki 의 retention.

---

## 6. log 도구 cost 비교

| | $/GB ingest | retention | query | 운영 |
| --- | --- | --- | --- | --- |
| **Datadog Logs** | $0.50 | $1.27/mo (15d) | included | SaaS |
| **CloudWatch Logs** | $0.50 | $0.03/mo | $0.005/GB | AWS |
| **Splunk** | $$$ | $$$ | included | 비쌈 |
| **Elastic Cloud** | $$ | $$ | included | 중간 |
| **Loki (self-host)** | $0 | S3 cheap | included | 운영 부담 |
| **OpenObserve** | $0 (OSS) | S3 cheap | included | 새 도구 |
| **Grafana Cloud Logs** | $0.50/M logs | included | included | SaaS |
| **Better Stack / Vector + S3** | 저렴 | S3 | Athena | hybrid |

→ 큰 회사 = Loki + Mimir self-host. 중간 = Grafana Cloud. 작은 = managed.

---

## 7. trace cost

```
distributed tracing:
  매 request × span × attribute = big data

Datadog APM:
  $$$ / host

Jaeger / Tempo (self-host):
  S3 storage = cheap
  단 운영

sampling:
  100%  → cost 폭주
  1%    → 일반 OK
  10%   → 깊이 분석
  100% for error 만
```

대응:
```
1. sampling rate ↓ (1% default)

2. tail-based sampling (★ 권장)
   - 모든 trace 임시 buffer
   - 끝나면 결정: error 면 keep, 정상 → drop
   - OTel Collector 가 처리

3. attribute 절제
   - 큰 string (full body) X
   - PII X
   - cardinality 가 metric 아니라 trace 도 영향

4. retention
   - hot 7d / cold S3 archive
```

---

## 8. agent / sidecar cost

```
service mesh (Istio sidecar):
  매 pod 에 Envoy = 50-100MB / pod
  100 pod = 5-10GB extra memory
  → node 더 필요 → cost

대안:
  - Cilium service mesh (eBPF, no sidecar)
  - Linkerd (가벼움)
  - 필요한 pod 만 inject
  
Datadog agent:
  매 host = $15/mo + 1 GB RAM
  → 100 host = $1500/mo

monitoring sidecar 의 noisy neighbor:
  CPU / memory steal
```

---

## 9. egress cost (★ hidden)

```
cross-region monitoring:
  - 본사 region 의 Datadog / Grafana
  - 다른 region 의 pod → egress $0.02/GB
  
1 PB / mo egress = $20,000/mo

대응:
  - region 별 self-host (local Loki)
  - aggregator (region 에서 sample → 본사)
  - VPC endpoint (Datadog 일부 지원)
  - 본사 region 에 SaaS endpoint
```

---

## 10. SaaS vs self-host (★ 결정)

```
SaaS (Datadog / Grafana Cloud):
  + 빠른 시작
  + 운영 부담 0
  + 통합 기능
  - 비용 매우 빠르게 증가
  - vendor lock-in

self-host (Prometheus + Loki + Tempo):
  + cost 1/10
  + control
  + customize
  - 운영 부담
  - 사람 시간 / on-call
  - feature 약간 적음

break-even:
  ~$10k/mo SaaS = self-host 운영자 1명 cost
  → 그 이상 = self-host 가 cheaper
```

---

## 11. self-host 표준 stack (★)

```
metric:
  Prometheus (각 cluster)
  → Mimir / Thanos (long-term, S3)
  
log:
  Vector / Fluent Bit (collector)
  → Loki (search) + S3 (archive)
  
trace:
  OTel SDK / agent
  → OTel Collector
  → Tempo (Grafana) + S3 (archive)
  
visualization:
  Grafana (모두 query)
  
alert:
  AlertManager → PagerDuty / Slack

storage:
  S3 / GCS object store (cheap)
  hot 14-30d local + cold S3
```

→ 한 회사가 stack 운영. 5-10 PB / yr 처리.

---

## 12. tiered storage

```
hot (local SSD):
  실시간 query
  7-14 day
  비쌈

warm (cheaper disk / object store):
  query 가끔
  30-90 day
  중간

cold (S3 Glacier):
  audit 만
  1+ year
  매우 cheap

도구:
  - Mimir (★ Grafana, S3 native)
  - Thanos (Prom + S3)
  - Loki (S3 native)
  - Tempo (S3 native)
  - Elastic ILM (Hot/Warm/Cold/Frozen)
```

→ "cheap object storage" 가 핵심.

---

## 13. monitoring 의 monitoring

```
누가 monitoring 의 비용 본다?
  
metric:
  - Prometheus 자체 의 cost
  - 폭증 시 alert
  
도구:
  - Grafana Mimir 의 self-monitoring
  - cost dashboard (per-team usage)
  - showback / chargeback

질문:
  - top 10 metric (by series)
  - top 10 log producer (by GB)
  - 안 보는 dashboard
  - 안 query 의 metric
  
→ 매월 review.
```

---

## 14. 절감 case study

### Datadog → Grafana stack migration

```
Before:
  Datadog $100k/mo
  
Migration:
  Prometheus + Loki + Tempo + Grafana
  + Mimir (long-term)
  
After:
  $10k/mo (infra)
  + $20k/mo (운영자 1명)
  = $30k/mo
  
saving: $70k/mo = $840k/yr

trade-off:
  - 6개월 마이그
  - feature 일부 약간
  - 운영 부담
```

### log 50% 감축

```
Before:
  100 GB/day = $1,500/mo (Datadog)

작업:
  - INFO → WARN
  - healthcheck filter
  - sampling 90% / 10% success
  - 큰 field 제거
  - retention 30d → 14d

After:
  20 GB/day = $300/mo

saving: $1,200/mo
```

---

## 15. log / metric / trace 의 ROI 평가

```
metric: 가장 cheap, 가장 query 많음
  → 더 늘려도 OK (단 cardinality 주의)

log: 비쌈, query 보통
  → 매우 절제 (level / filter / retention)

trace: 매우 비싸, query 적음
  → sampling 1% + tail-based for error

→ cost ≠ value. 신중한 cost-value 평가.
```

---

## 16. 함정

1. **모든 것 capture** — 비용 폭주.
2. **retention 무한** — S3 도 영원하면 비쌈.
3. **DEBUG in production** — 큰 log volume.
4. **PII in log** — GDPR + storage cost.
5. **dashboard 무관심** — 안 쓰는 panel 가 매일 query.
6. **cardinality 모니터링 안 함** — silent 폭증.
7. **agent / sidecar 비용 무시** — pod 마다 50MB.
8. **multi-region 의 egress** — 본사 monitoring SaaS.

---

## 17. quarterly review check

```
☐ top 10 metric by series count
☐ top 10 log source by GB
☐ unused dashboard / alert
☐ retention 정책 review
☐ scrape interval 적정
☐ sampling rate 적정
☐ agent / sidecar 부담
☐ team 별 cost (showback)
☐ ROI (cost vs value)
☐ vendor invoice 분석
```

---

## 18. 관련

- [[monitoring|↑ monitoring]]
- [[cardinality-management]]
- [[../finops/finops|↗ FinOps]]
- [[../finops/architecture-optimization|↗ architecture cost]]
