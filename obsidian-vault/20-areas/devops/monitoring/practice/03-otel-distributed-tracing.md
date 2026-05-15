---
title: "실습 03 — OTel + Tempo distributed tracing"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T06:00:00+09:00
tags: [devops, monitoring, practice, opentelemetry, tempo]
---

# 실습 03 — OTel + Tempo distributed tracing

**[[practice|↑ practice]]**

---

## 목표

- OTel Java agent 로 Spring app trace 자동 수집
- OTel Collector → Tempo
- Grafana 에서 trace waterfall
- Loki log ↔ Tempo trace 점프 (exemplar)

---

## 1. Tempo 설치

```bash
helm install tempo grafana/tempo -n monitoring
```

---

## 2. OTel Collector 설치

```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts

helm install otel-collector open-telemetry/opentelemetry-collector \
    -n monitoring \
    --set mode=deployment \
    --values otel-values.yaml
```

```yaml
# otel-values.yaml
config:
  receivers:
    otlp:
      protocols:
        grpc: {endpoint: 0.0.0.0:4317}
        http: {endpoint: 0.0.0.0:4318}
  exporters:
    otlp/tempo:
      endpoint: tempo:4317
      tls: {insecure: true}
    prometheus:
      endpoint: 0.0.0.0:8889
  service:
    pipelines:
      traces:  {receivers: [otlp], exporters: [otlp/tempo]}
      metrics: {receivers: [otlp], exporters: [prometheus]}
```

---

## 3. Spring 배포 — OTel agent 추가

```dockerfile
FROM eclipse-temurin:21-jre-jammy
WORKDIR /app
ADD https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar /agent.jar
COPY app.jar /app/app.jar
ENTRYPOINT ["java", "-javaagent:/agent.jar", "-jar", "/app/app.jar"]
```

```yaml
# deployment.yaml (env 추가)
env:
  - {name: OTEL_SERVICE_NAME, value: demo-web}
  - {name: OTEL_EXPORTER_OTLP_ENDPOINT, value: http://otel-collector.monitoring:4317}
  - {name: OTEL_EXPORTER_OTLP_PROTOCOL, value: grpc}
  - {name: OTEL_TRACES_SAMPLER, value: parentbased_traceidratio}
  - {name: OTEL_TRACES_SAMPLER_ARG, value: "1.0"}    # 실습: 100%, prod: 0.01
  - {name: OTEL_LOGS_EXPORTER, value: none}
```

---

## 4. log 에 trace_id 자동 포함

```yaml
# application.yml
logging:
  pattern:
    level: "%5p [%X{trace_id:-},%X{span_id:-}]"
```

또는 logback-spring.xml 의 MDC encoder 에 `trace_id`, `span_id` 포함 (실습 02 와 동일).

→ OTel agent 가 자동으로 SLF4J MDC 에 trace_id 주입.

---

## 5. 멀티 서비스 — 2개 서비스 추가

```kotlin
// demo-web 의 controller — 다른 서비스 호출
@RestController
class OrderController(private val client: RestTemplate) {
    @GetMapping("/order")
    fun createOrder(): String {
        val payment = client.getForObject(
            "http://demo-payment.app:8080/charge", String::class.java
        )
        return "order ok, payment=$payment"
    }
}
```

```kotlin
// demo-payment 의 controller
@RestController
class PaymentController {
    @GetMapping("/charge")
    fun charge(): String {
        Thread.sleep(50)        // 의도적 latency
        return "paid"
    }
}
```

→ 두 서비스 모두 OTel agent 적용. trace_id 가 W3C header 로 자동 전파.

---

## 6. 부하 주입

```bash
kubectl run -it --rm load --image=williamyeh/hey -- \
    hey -z 60s -c 5 http://demo-web.app:8080/order
```

---

## 7. Grafana — Tempo Data source

```
Add Data source → Tempo
URL: http://tempo.monitoring:3100
Save & test
```

→ Loki Data source 의 "Derived fields" 에서:
```
Name: trace_id
Regex: trace_id=(\w+)
URL: ${__value.raw}
Internal link: Tempo
```

→ Loki log 에 trace_id 가 link 로 변환됨.

---

## 8. trace waterfall 확인

```
Grafana → Explore → Tempo → Search
Service Name: demo-web
Min Duration: 0ms
Run query
```

trace 선택 → waterfall:
```
demo-web GET /order              [120ms]
  └─ http.client demo-payment    [60ms]
      └─ demo-payment GET /charge [55ms]
          └─ Thread.sleep         [50ms]
```

---

## 9. log → trace 점프 (★)

```
1. Grafana Explore → Loki
2. {pod=~"demo-web-.*"} |= "ERROR"
3. log 한 줄 클릭 → trace_id 옆에 Tempo 버튼 표시
4. 클릭 → 해당 request 의 전체 trace 점프
```

---

## 10. sampling 조정 (prod 시나리오)

```yaml
# 100% → 1%
- {name: OTEL_TRACES_SAMPLER_ARG, value: "0.01"}
```

→ OTel Collector 의 tail-based sampling 으로 "error 만 100%" 도 가능.

---

## 11. 검증 체크리스트

- [ ] Tempo Data source 정상
- [ ] demo-web → demo-payment 호출 시 한 trace 안에 2 service span
- [ ] log 에 `[trace_id,span_id]` 패턴 포함
- [ ] Loki log 클릭 → Tempo trace 점프
- [ ] sampling 1% 변경 후 trace 수 ↓ 확인

---

## 관련

- [[practice|↑ practice]]
- [[../opentelemetry]]
- [[../jaeger-tempo]]
- [[02-loki-log-aggregation]]
