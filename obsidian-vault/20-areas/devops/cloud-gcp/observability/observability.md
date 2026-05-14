---
title: "GCP Observability — Cloud Logging / Monitoring"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:55:00+09:00
tags:
  - gcp
  - observability
  - logging
  - monitoring
---

# GCP Observability

**[[../cloud-gcp|↑ GCP]]**

---

## 1. 구성 (Operations Suite, 옛 Stackdriver)

| 영역 | 의미 |
| --- | --- |
| **Cloud Logging** | log (모든 GCP 자원 자동) |
| **Cloud Monitoring** | metric / alert / dashboard |
| **Cloud Trace** | 분산 trace |
| **Cloud Profiler** | continuous profiling |
| **Cloud Debugger** | (deprecated) |
| **Error Reporting** | exception aggregation |

→ AWS CloudWatch + X-Ray + Error 비슷.

---

## 2. Cloud Logging

자동:
- 모든 GCP 자원 (Cloud Run / GKE / GCE / Functions / ...)
- Audit Logs (Admin / Data Access / System Event)
- VPC Flow Logs

```bash
gcloud logging read 'resource.type="cloud_run_revision" AND severity>=ERROR' --limit=20

# 응용 — stdout 출력 + 자동 capture
print('{"severity":"INFO","message":"req","path":"/","status":200}')
```

JSON 출력 = 자동 structured.

### 2.1 Log Sink

```hcl
resource "google_logging_project_sink" "to_bq" {
  name        = "audit-to-bq"
  destination = "bigquery.googleapis.com/projects/PROJECT/datasets/audit"
  filter      = "logName:\"cloudaudit.googleapis.com\""
}
```

→ log 를 BigQuery / GCS / Pub/Sub 으로 export.

---

## 3. Cloud Monitoring

자동 metric — `compute.googleapis.com/instance/cpu/utilization` 등.

### 3.1 Custom metric

```python
from google.cloud import monitoring_v3
client = monitoring_v3.MetricServiceClient()
series = monitoring_v3.TimeSeries()
series.metric.type = "custom.googleapis.com/orders/processed"
series.points.add(value={"int64_value": 1}, interval={"end_time": ...})
client.create_time_series(name=..., time_series=[series])
```

### 3.2 Alert Policy

```hcl
resource "google_monitoring_alert_policy" "high_cpu" {
  display_name = "High CPU"
  combiner     = "OR"

  conditions {
    display_name = "CPU > 80"
    condition_threshold {
      filter          = "resource.type=\"gce_instance\" AND metric.type=\"compute.googleapis.com/instance/cpu/utilization\""
      duration        = "300s"
      comparison      = "COMPARISON_GT"
      threshold_value = 0.8
      aggregations {
        alignment_period   = "60s"
        per_series_aligner = "ALIGN_MEAN"
      }
    }
  }

  notification_channels = [google_monitoring_notification_channel.email.id]
}
```

---

## 4. Cloud Trace

OpenTelemetry / OpenCensus 자동 수집. 분산 trace 시각화.

Cloud Run / GKE 자동.

---

## 5. 비용

```
Cloud Logging:    50 GB / project 무료, 이후 $0.50 / GB ingest
Cloud Monitoring: 150 MB metric 무료, 이후 $0.258 / MB
Cloud Trace:      2.5M span 무료, 이후 $0.20 / 1M
```

→ log retention = 30일 default. 더 길면 cost ↑. Log sink → GCS / BigQuery archive.

---

## 6. 함정

- audit log 의 Data Access 활성 = 비용 폭증
- 응용이 매우 verbose 한 log
- log retention = 30일 default
- VPC Flow Logs 활성 시 cost
- alert 의 cooldown 신중

---

## 7. 외부

- **Grafana** + Cloud Monitoring datasource
- **Datadog / NewRelic** APM (vendor agnostic)
- **Prometheus** on GKE + Managed Prometheus

---

## 8. 관련

- [[../cloud-gcp|↑ GCP]]
- [[../security/iam]] — audit
- [[../../monitoring/monitoring]]
