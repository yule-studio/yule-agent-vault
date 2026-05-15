---
title: "OpenTelemetry — 표준 instrumentation ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:40:00+09:00
tags: [devops, monitoring, opentelemetry, otel]
---

# OpenTelemetry — 표준 instrumentation ★

**[[monitoring|↑ monitoring]]**

---

## 1. 무엇

- **metric + log + trace** 통합 표준 (CNCF).
- vendor-neutral (Prometheus / Datadog / Jaeger 모두 호환).
- SDK (Java / Python / Go / Node / .NET / Ruby).

---

## 2. 흐름

```
[App + OTel SDK] → OTLP → OTel Collector → exporter
                                          → Prometheus / Loki / Tempo
                                          → Datadog / NewRelic / Jaeger
```

---

## 3. Spring Boot 통합

```kotlin
// build.gradle
implementation("io.opentelemetry:opentelemetry-api")
implementation("io.opentelemetry.instrumentation:opentelemetry-spring-boot-starter")
```

```bash
# 또는 agent (zero-code)
java -javaagent:opentelemetry-javaagent.jar \
     -Dotel.service.name=my-app \
     -Dotel.exporter.otlp.endpoint=http://otel-collector:4317 \
     -jar app.jar
```

→ 자동 trace + metric (Spring / JDBC / Kafka / Redis 등).

---

## 4. OTel Collector

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc: {endpoint: 0.0.0.0:4317}
      http: {endpoint: 0.0.0.0:4318}

processors:
  batch:
    timeout: 10s
  memory_limiter:
    limit_mib: 1024

exporters:
  prometheus: {endpoint: 0.0.0.0:8889}
  loki: {endpoint: http://loki:3100/loki/api/v1/push}
  otlp/tempo:
    endpoint: http://tempo:4317
    tls: {insecure: true}

service:
  pipelines:
    metrics: {receivers: [otlp], processors: [batch], exporters: [prometheus]}
    logs:    {receivers: [otlp], processors: [batch], exporters: [loki]}
    traces:  {receivers: [otlp], processors: [batch], exporters: [otlp/tempo]}
```

---

## 5. Trace 예시

```java
// 자동 — agent 시
@RestController
public class MyController {
    @GetMapping("/api")
    public String get() {
        return service.process();        // trace 자동
    }
}

// 수동 span
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.Tracer;

Tracer tracer = openTelemetry.getTracer("my-app");
Span span = tracer.spanBuilder("process-order").startSpan();
try (var scope = span.makeCurrent()) {
    // ... work
} finally {
    span.end();
}
```

---

## 6. 함정

1. **agent + manual instrumentation 동시** → 중복 span.
2. **sampling rate 100%** → 비용 폭주 → 1% 또는 head-based / tail-based.
3. **너무 많은 attribute** → cardinality 폭주.
4. **OTel Collector 자체 monitoring** 안 함 → blind spot.
5. **OTLP gRPC vs HTTP** — 방화벽 / port.

---

## 7. 관련

- [[monitoring|↑ monitoring]]
- [[jaeger-tempo]]
- [[prometheus]]
- [[loki]]
