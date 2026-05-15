---
title: "Jaeger / Tempo — distributed tracing"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:42:00+09:00
tags: [devops, monitoring, tracing, jaeger, tempo]
---

# Jaeger / Tempo — distributed tracing

**[[monitoring|↑ monitoring]]**

---

## 1. 왜 필요

- MSA 환경에서 한 request 가 여러 서비스를 거침.
- log 만으로는 흐름 추적 어려움 (correlation-id 도 manual).
- trace = 한 request 의 전체 path + 각 span latency.

---

## 2. 무엇

| | 무엇 |
| --- | --- |
| **Jaeger** | Uber 개발, 기존 표준 (OpenTracing 시절) |
| **Tempo** | Grafana Labs, object storage 기반 (저비용) |
| **Zipkin** | Twitter 개발, 가장 오래된 OSS |
| **AWS X-Ray** | AWS 통합 |
| **Datadog APM** | SaaS |

**현재 권장**: OTel → Tempo (Grafana 통합) 또는 Jaeger.

---

## 3. Tempo 설치 (Helm)

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm install tempo grafana/tempo -n monitoring
```

→ Grafana Data source 에서 Tempo 추가 (URL: `http://tempo:3100`).

---

## 4. Trace 흐름

```
[Client] → service-A → service-B → DB
              │           │
              ▼           ▼
        trace_id=abc  trace_id=abc (전파)
        span_id=1     span_id=2 (parent=1)
```

→ TraceContext header (W3C): `traceparent: 00-<trace_id>-<span_id>-01`.

---

## 5. Grafana 에서 trace 보기

```
1. Loki 에서 log 검색 → trace_id 발견
2. trace_id 클릭 → Tempo data source 로 점프
3. waterfall (span timeline) 확인
4. slow span 식별 → 해당 span 의 tag (db.query, http.url) 확인
```

→ **log ↔ trace ↔ metric** 연결 (exemplar).

---

## 6. Spring Boot — OTel agent (자동)

```bash
java -javaagent:opentelemetry-javaagent.jar \
     -Dotel.exporter.otlp.endpoint=http://otel-collector:4317 \
     -Dotel.service.name=order-service \
     -jar app.jar
```

→ HTTP / JDBC / Kafka / Redis 자동 trace.

---

## 7. sampling 전략

| 전략 | 무엇 | 언제 |
| --- | --- | --- |
| **always_on** | 100% | dev / staging |
| **probabilistic** | 1-10% | 정상 prod (low traffic) |
| **rate-limiting** | 초당 N개 | high traffic |
| **head-based** | request 시작 시 결정 | 일반적 |
| **tail-based** | trace 완료 후 결정 (error 만 등) | Collector 에서, 비용 ↑ |

---

## 8. 함정

1. **trace_id 전파 안 됨** — Kafka producer/consumer 에서 header 안 넘김.
2. **sampling 100%** → 비용 폭주 (특히 prod).
3. **span attribute 과다** — PII / huge payload.
4. **storage** — Tempo 는 S3 / GCS object storage 권장 (디스크 비쌈).
5. **trace ↔ log 연결 안 됨** — log 에 trace_id 안 찍힘 (MDC 설정 필요).

---

## 9. 관련

- [[monitoring|↑ monitoring]]
- [[opentelemetry]]
- [[loki]]
- [[grafana]]
