---
title: "Application metrics — RED / USE / Golden Signals"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:50:00+09:00
tags: [devops, monitoring, metrics, red, use]
---

# Application metrics — RED / USE / Golden Signals

**[[monitoring|↑ monitoring]]**

---

## 1. 3가지 방법론

| 방법론 | 적용 | 핵심 |
| --- | --- | --- |
| **RED** | request-based (API / service) | Rate, Errors, Duration |
| **USE** | resource-based (CPU/Mem/Disk/Net) | Utilization, Saturation, Errors |
| **Golden Signals** | Google SRE 통합 | Latency, Traffic, Errors, Saturation |

→ **RED + USE = 거의 모든 시스템 커버**.

---

## 2. RED method (Weaveworks)

| | 무엇 | 예 |
| --- | --- | --- |
| **R** ate | 초당 request | `rate(http_total[1m])` |
| **E** rrors | 실패율 | `rate(http_5xx[1m]) / rate(http_total[1m])` |
| **D** uration | 응답 시간 | `histogram_quantile(0.95, http_duration_bucket)` |

→ 사용자 경험과 직결.

---

## 3. USE method (Brendan Gregg)

| | 무엇 | 예 |
| --- | --- | --- |
| **U** tilization | 사용률 | CPU 80%, Mem 60% |
| **S** aturation | 큐 대기 / 폭주 | run-queue length, GC pause |
| **E** rrors | 하드웨어 / OS error | disk read error, OOM kill |

→ resource (CPU/Mem/Disk/Net) 마다 측정.

---

## 4. Golden Signals (Google SRE Book)

| | 무엇 |
| --- | --- |
| **Latency** | 응답 시간 (성공 vs 실패 분리) |
| **Traffic** | request rate / qps |
| **Errors** | failed request rate |
| **Saturation** | 시스템 포화 (queue, thread pool) |

---

## 5. Spring Boot Actuator + Micrometer

```kotlin
// build.gradle
implementation("org.springframework.boot:spring-boot-starter-actuator")
implementation("io.micrometer:micrometer-registry-prometheus")
```

```yaml
# application.yml
management:
  endpoints.web.exposure.include: prometheus,health,info,metrics
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true
      sla:
        http.server.requests: 100ms,500ms,1s
    tags:
      application: ${spring.application.name}
      env: ${ENV:dev}
```

→ `GET /actuator/prometheus` 노출.

---

## 6. 기본 노출되는 metric

| Category | metric |
| --- | --- |
| **HTTP** | `http_server_requests_seconds_*` |
| **JVM** | `jvm_memory_used_bytes`, `jvm_gc_pause_seconds`, `jvm_threads_live` |
| **Process** | `process_cpu_usage`, `system_cpu_usage` |
| **DataSource** | `hikaricp_connections_*` (HikariCP) |
| **Cache** | `cache_gets_total`, `cache_size` |
| **Tomcat** | `tomcat_threads_*`, `tomcat_sessions_*` |
| **Kafka** | `kafka_consumer_*`, `kafka_producer_*` |

---

## 7. 커스텀 metric

```kotlin
@Component
class OrderMetrics(meterRegistry: MeterRegistry) {

    // Counter
    private val orderCreated = Counter.builder("orders.created")
        .description("주문 생성 수")
        .tag("source", "web")
        .register(meterRegistry)

    // Gauge
    private val pendingOrders = AtomicInteger(0)
    init {
        Gauge.builder("orders.pending", pendingOrders) { it.get().toDouble() }
            .register(meterRegistry)
    }

    // Timer
    private val orderProcessTime = Timer.builder("orders.process.duration")
        .publishPercentiles(0.5, 0.95, 0.99)
        .register(meterRegistry)

    fun recordCreated() = orderCreated.increment()

    fun <T> recordProcess(action: () -> T): T = orderProcessTime.recordCallable(action)!!
}
```

---

## 8. 비즈니스 metric (KPI)

```kotlin
// 매출
orderTotal.increment(order.amount.toDouble(), Tag.of("channel", channel))

// 결제 성공률
paymentSuccess.increment()  // 또는 fail
// → rate(payment_success_total) / rate(payment_total)
```

→ Grafana → business dashboard.

---

## 9. cardinality 주의

```kotlin
// ❌ user_id 같은 high cardinality
.tag("user_id", userId)         // 백만 명 → 백만 series → OOM

// ✅ 그룹 단위
.tag("user_tier", "premium")
```

→ tag 는 **유한한 값 집합** 만.

---

## 10. 함정

1. **percentile 평균** — `avg(p95)` 는 수학적 오류 → `histogram_quantile`.
2. **success-only metric** — error 비율 계산 불가.
3. **tag 누락** — `app`, `env`, `version` 표준화.
4. **gauge 만 사용** — counter (`_total`) 가 rate 가능.
5. **business metric 분리** — 시스템 metric 과 같은 곳에 두면 lost.

---

## 11. 관련

- [[monitoring|↑ monitoring]]
- [[prometheus]]
- [[slos-and-sli]]
- [[grafana]]
