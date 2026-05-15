---
title: "Grafana — 시각화"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:34:00+09:00
tags: [devops, monitoring, grafana]
---

# Grafana — 시각화

**[[monitoring|↑ monitoring]]**

---

## 1. 무엇

- multi-source 대시보드 (Prometheus / Loki / Tempo / Elastic / MySQL / Postgres / etc.).
- alert (UI 또는 alerting plugin).
- 대시보드 = JSON / provision.

---

## 2. data source 추가 (Prometheus)

```
Configuration → Data sources → Add → Prometheus
URL: http://prometheus.monitoring:9090
Save & test
```

---

## 3. 대시보드 example panel

```
title: "HTTP RPS by status"
query: sum by (status) (rate(http_server_requests_seconds_count{app="web"}[1m]))
viz: time series
```

---

## 4. 대시보드 as code (provision)

```yaml
# /etc/grafana/provisioning/dashboards/dashboards.yml
apiVersion: 1
providers:
  - name: 'default'
    folder: 'Apps'
    type: file
    options:
      path: /var/lib/grafana/dashboards
```

```json
// dashboards/web.json
{ "title": "Web", "panels": [...] }
```

---

## 5. 표준 대시보드

| 종류 | 무엇 |
| --- | --- |
| Golden Signals | RPS / Latency / Error rate / Saturation |
| RED | per-service |
| USE | per-resource |
| SLO dashboard | error budget / burn rate |
| business | 매출 / 결제 / 사용자 |

---

## 6. variable (template)

```
Dashboard settings → Variables
$namespace = label_values(kube_pod_info, namespace)
$pod = label_values(kube_pod_info{namespace="$namespace"}, pod)

panel query: rate(container_cpu_usage_seconds_total{namespace="$namespace", pod="$pod"}[5m])
```

---

## 7. Grafana Cloud (관리형)

- self-host Grafana + Mimir / Loki / Tempo 운영 부담 ↓.
- Free tier (1 user / 50GB log / 10k series).
- Pro $$ — 큰 규모.

---

## 8. 함정

1. **대시보드 너무 많음** — folder / tag.
2. **무거운 query** — time range / interval.
3. **변수 dependency** — 잘못된 default.
4. **annotation overload** — deploy / incident annotation.
5. **TimescaleDB vs Prometheus** — query latency 차이.

---

## 9. 관련

- [[monitoring|↑ monitoring]]
- [[prometheus]]
- [[loki]]
- [[alerting]]
