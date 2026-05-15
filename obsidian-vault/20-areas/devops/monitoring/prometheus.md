---
title: "Prometheus ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:32:00+09:00
tags: [devops, monitoring, prometheus]
---

# Prometheus ★

**[[monitoring|↑ monitoring]]**

---

## 1. 핵심

- pull 모델 (Prometheus 가 target HTTP scrape).
- TSDB (time-series database).
- PromQL.
- service discovery (k8s / consul / DNS).

---

## 2. k8s 설치 (kube-prometheus-stack)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kps prometheus-community/kube-prometheus-stack \
    -n monitoring --create-namespace \
    --set grafana.adminPassword=secret
```

→ Prometheus + Grafana + AlertManager + node-exporter + kube-state-metrics 자동.

---

## 3. Spring Boot — Micrometer

```kotlin
// build.gradle
implementation("io.micrometer:micrometer-registry-prometheus")
```

```yaml
# application.yml
management:
  endpoints.web.exposure.include: prometheus,health
  metrics.tags.application: ${spring.application.name}
```

→ `GET /actuator/prometheus` 로 metric 노출.

```yaml
# Prometheus scrape config (또는 ServiceMonitor CRD)
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata: {name: web}
spec:
  selector: {matchLabels: {app: web}}
  endpoints:
    - port: http
      path: /actuator/prometheus
      interval: 15s
```

---

## 4. PromQL 기본

```promql
# CPU usage by Pod
sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="prod"}[5m]))

# HTTP request rate
sum by (method, status) (rate(http_server_requests_seconds_count{app="web"}[1m]))

# 95th percentile latency
histogram_quantile(0.95,
    sum by (le, method) (rate(http_server_requests_seconds_bucket{app="web"}[5m]))
)

# Error rate
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
/ sum(rate(http_server_requests_seconds_count[5m]))
```

---

## 5. metric 4가지

| Type | 예 |
| --- | --- |
| **Counter** | `http_requests_total` — 증가만 |
| **Gauge** | `cpu_usage`, `queue_size` — 증가/감소 |
| **Histogram** | `http_duration_bucket` + sum + count |
| **Summary** | percentile pre-computed (Prom 권장 X — histogram_quantile 사용) |

---

## 6. RED / USE method

| Method | 적용 | 핵심 |
| --- | --- | --- |
| **RED** | request 기반 (API) | Rate, Errors, Duration |
| **USE** | resource 기반 (CPU/Memory/Disk) | Utilization, Saturation, Errors |

---

## 7. long-term storage

- Prometheus 자체: 15일 default.
- **Thanos** — object storage (S3) 통합.
- **Cortex / Mimir** — multi-tenant + scale.

---

## 8. 함정

1. **scrape 한도 (10s)** — 너무 자주 → 부하.
2. **high cardinality label** (user_id, path) → cardinality 폭주 → OOM.
3. **PromQL 의 `sum / by`** — label 명시 안 하면 모든 label 별 분리.
4. **histogram bucket 잘못** → percentile 부정확.
5. **drop label rule** — 너무 많은 metric → 비용.

---

## 9. 관련

- [[monitoring|↑ monitoring]]
- [[grafana]]
- [[alerting]]
- [[application-metrics]]
