---
title: "Monitoring / Observability hub"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:28:00+09:00
tags: [devops, monitoring, observability, hub]
---

# Monitoring / Observability hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[../devops|↑ devops]]**

---

## 0. 3 pillar of observability

| Pillar | 무엇 | 도구 |
| --- | --- | --- |
| **Metrics** | 수치 (RPS / latency / CPU) | Prometheus / Datadog |
| **Logs** | 텍스트 (event / error) | Loki / ELK / Splunk |
| **Traces** | 분산 request 의 단계 | Jaeger / Tempo / Datadog APM |

→ 통합: **OpenTelemetry** 표준.

---

## 1. 영역

| 노트 | 내용 |
| --- | --- |
| [[tools-comparison]] ★ | Prometheus / Datadog / Grafana Cloud / New Relic |
| [[prometheus]] ★ | metric 표준 |
| [[grafana]] ★ | 시각화 |
| [[loki]] | log (lightweight) |
| [[elk-stack]] | Elasticsearch + Logstash + Kibana |
| [[opentelemetry]] ★ | metric + log + trace 통합 표준 |
| [[jaeger-tempo]] | distributed tracing |
| [[alerting]] ★ | AlertManager / PagerDuty / Opsgenie |
| [[slos-and-sli]] | service level objective |
| [[application-metrics]] | RED / USE method |
| [[pitfalls]] | 흔한 함정 |
| [[practice/practice]] ★ | 실습 |

---

## 2. 도구 비교 요약

| | OSS | SaaS | 비용 |
| --- | --- | --- | --- |
| **Prometheus + Grafana** ★ | O | (Grafana Cloud) | low |
| **Datadog** | X | O | 비쌈 ($15-23/host) |
| **New Relic** | X | O | 비쌈 |
| **CloudWatch / Stackdriver / Azure Monitor** | X | O (cloud) | 중 |
| **Dynatrace** | X | O | 비쌈 (enterprise) |
| **ELK** | O | O (Elastic Cloud) | 중-high |
| **Loki + Grafana** | O | (Grafana Cloud) | low |
| **OpenTelemetry + Tempo** | O | (Grafana Cloud) | low |

→ 시작: Prometheus + Grafana + Loki (self-host) 또는 Grafana Cloud (관리형).

---

## 3. 본 vault 권장

```
시작:
  Prometheus + Grafana + Loki (또는 Grafana Cloud)
  + Spring Boot Actuator + Micrometer
  + Slack alert (단순)

성장:
  + AlertManager + on-call rotation (PagerDuty / Opsgenie)
  + OpenTelemetry SDK
  + Jaeger / Tempo (분산 trace)
  + SLO/SLI 정의 + error budget

대형 / enterprise:
  + Datadog or New Relic (통합 SaaS)
  + custom dashboard per team
```

---

## 4. 관련

- [[../devops|↑ devops]]
- [[../sre/sre|↗ sre — SLO / incident]]
- [[../kubernetes/autoscaling|↗ HPA]]
