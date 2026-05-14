---
title: "OCI Observability — Monitoring / Logging"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:45:00+09:00
tags:
  - oci
  - observability
  - monitoring
  - logging
  - hub
---

# OCI Observability

**[[../cloud-oci|↑ OCI]]**

---

## 1. 구성

| 영역 | 의미 |
| --- | --- |
| **Monitoring** | metric (CloudWatch 동등) |
| **Logging** | log (CloudWatch Logs) |
| **Logging Analytics** | log 분석 (Splunk-like, 별도) |
| **APM** (Application Performance Monitoring) | trace / span |
| **Events Service** | OCI 자원 이벤트 라우팅 |
| **Notifications** | alarm subscriber |

---

## 2. Monitoring (Metric)

```bash
oci monitoring metric-data summarize-metrics-data \
  --compartment-id <ocid> \
  --namespace oci_computeagent \
  --query-text 'CpuUtilization[1m].mean()'
```

```hcl
resource "oci_monitoring_alarm" "high_cpu" {
  compartment_id        = var.compartment_ocid
  display_name          = "high-cpu"
  is_enabled            = true
  metric_compartment_id = var.compartment_ocid
  namespace             = "oci_computeagent"
  query                 = "CpuUtilization[1m].mean() > 80"
  severity              = "CRITICAL"
  destinations          = [oci_ons_notification_topic.alerts.id]
  pending_duration      = "PT5M"
}
```

---

## 3. Logging

3 가지 source:

| | |
| --- | --- |
| **Service log** | OCI 서비스 자체 (LB / API GW / FN ...) |
| **Custom log** | 응용이 push (Agent / Fluentd) |
| **Audit log** | tenancy 의 모든 API call (CloudTrail 동등, 자동) |

```hcl
resource "oci_logging_log_group" "app" {
  compartment_id = var.compartment_ocid
  display_name   = "app-logs"
}

resource "oci_logging_log" "lb_access" {
  display_name = "lb-access"
  log_group_id = oci_logging_log_group.app.id
  log_type     = "SERVICE"

  configuration {
    source {
      category    = "access"
      resource    = oci_load_balancer_load_balancer.lb.id
      service     = "loadbalancer"
      source_type = "OCISERVICE"
    }
  }
}
```

---

## 4. Logging Search

```
search "myapp-tenancy/app-logs/lb-access"
| where data.request.method = 'POST'
| sort by datetime desc
| top 100
```

→ web UI 기반. KQL 만큼은 아님.

---

## 5. APM (Distributed Tracing)

```bash
# APM domain 생성
oci apm-control-plane apm-domain create ...

# OpenTelemetry agent (Java / Python / Node)
java -javaagent:apm-agent.jar -Dcom.oracle.apm.agent.service.name=myapp -jar app.jar
```

→ span / trace / metric 자동.

---

## 6. Service Connector (Stream / Function)

```hcl
resource "oci_sch_service_connector" "logs_to_storage" {
  source { kind = "logging"; log_sources { ... } }
  target { kind = "objectStorage"; bucket = "logs-archive" }
}
```

→ log 를 Object Storage 로 archive / Stream / Function 으로.

---

## 7. Audit Log = AWS CloudTrail

모든 control plane API 자동 — 365 일 보관 (default), Storage 로 archive 가능.

---

## 8. 가격

```
Monitoring:  대부분 무료 (기본 metric / alarm)
Logging:    $0.05 / GB ingest + $0.05 / GB·month storage
APM:        $4 / 1M span
Logging Analytics: 별도 (비쌈)
```

→ AWS / Azure 보다 저렴.

---

## 9. 외부

- **Grafana OCI plugin**
- **Datadog OCI integration**
- **Prometheus / OpenTelemetry** Push to APM

---

## 10. 관련

- [[../cloud-oci|↑ OCI]]
- [[../../monitoring/monitoring]]
- [[../messaging/notifications]] — alarm subscriber
