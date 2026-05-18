---
title: "monitoring/actuator-micrometer — Spring Actuator + Micrometer"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-19T07:30:00+09:00
tags: [backend, java-spring, observability, monitoring, actuator, micrometer]
home_hub: monitoring
related:
  - "[[monitoring]]"
  - "[[prometheus-grafana]]"
  - "[[../logging/elk-stack]]"
---

# monitoring/actuator-micrometer — Spring Actuator + Micrometer

**[[monitoring|↑ monitoring]]** · **[[../observability|↑ observability]]**

> 강의 매핑: §7 (Actuator / Micrometer / 메트릭 노출).

---

## 1. 목적

본 문서는 Spring Boot 응용의 **메트릭 노출** — Spring Actuator endpoint + Micrometer registry — 를 정의한다.

본 문서가 정의하는 것:
- Spring Actuator 의 표준 endpoint 16+ (health / info / metrics / prometheus / loggers / env / heapdump 등)
- Micrometer 의 4 instrument (Counter / Gauge / Timer / DistributionSummary)
- Spring 자동 측정 (HTTP / JVM / 시스템 / DataSource / Hikari)
- 사용자 정의 메트릭 작성
- Prometheus 포맷 export (`/actuator/prometheus`)
- 보안 설정 (production endpoint 보호)

본 문서가 정의하지 않는 것:
- Prometheus 자체 설치 / PromQL — [[prometheus-grafana]]
- Grafana 대시보드 - [[prometheus-grafana]]
- 로그 (Logback / ELK) — [[../logging/elk-stack]]

---

## 2. Spring Actuator 의 정의

Spring Boot 의 **운영 정보 노출 endpoint** 라이브러리. health check / 메트릭 / 환경변수 / heap dump / 로그 level 등 운영에 필요한 정보를 HTTP / JMX 로 제공.

### 2.1 의존성

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
}
```

### 2.2 기본 활성화 — `/actuator`

```bash
curl http://localhost:8080/actuator
{
  "_links": {
    "self": { "href": "http://localhost:8080/actuator" },
    "health": { "href": "http://localhost:8080/actuator/health" },
    "info":   { "href": "http://localhost:8080/actuator/info" }
  }
}
```

→ 기본은 `health` + `info` 만 노출. 다른 endpoint 는 명시적 활성화.

### 2.3 endpoint 노출 설정

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,loggers,env,httpexchanges
        exclude: ""           # 또는 특정 제외
      base-path: /actuator    # default

  endpoint:
    health:
      show-details: when-authorized      # always / never / when-authorized
      probes:
        enabled: true                     # k8s liveness / readiness
    prometheus:
      enabled: true                       # /actuator/prometheus 활성
    env:
      show-values: when-authorized

  server:
    port: 8081                            # ★ 별도 port — production 권장 (보안)
    address: 127.0.0.1                    # localhost 만 (private)
```

→ `management.server.port=8081` 분리는 production 의 표준. main port (8080) 는 외부, 8081 은 internal 만.

---

## 3. 표준 endpoint

### 3.1 health — k8s probe + uptime monitor

```bash
curl http://localhost:8080/actuator/health
{ "status": "UP" }

curl http://localhost:8080/actuator/health -H "Authorization: Bearer ..."
{
  "status": "UP",
  "components": {
    "db":            { "status": "UP", "details": { "database": "PostgreSQL", "validationQuery": "isValid()" } },
    "diskSpace":     { "status": "UP", "details": { "total": 250GB, "free": 100GB, "threshold": 10MB } },
    "redis":         { "status": "UP", "details": { "version": "7.2.0" } },
    "ping":          { "status": "UP" },
    "livenessState": { "status": "UP" },
    "readinessState":{ "status": "UP" }
  }
}
```

| 항목 | 사용 |
| --- | --- |
| `/actuator/health` | 통합 status |
| `/actuator/health/liveness` | k8s liveness probe (재시작 결정) |
| `/actuator/health/readiness` | k8s readiness probe (트래픽 라우팅) |

### 3.2 info — 빌드 / 버전 정보

```yaml
# application.yml
management:
  info:
    env:
      enabled: true
    git:
      enabled: true
      mode: full
    build:
      enabled: true

info:
  app:
    name: ${spring.application.name}
    version: ${app.version:unknown}
```

```bash
curl http://localhost:8080/actuator/info
{
  "app":   { "name": "restaurant-api", "version": "1.2.3" },
  "git":   { "branch": "main", "commit": { "id": "a7c3...", "time": "2026-05-19T08:00:00Z" } },
  "build": { "artifact": "restaurant-api", "name": "restaurant-api", "time": "2026-05-19T08:30:00Z", "version": "1.2.3" }
}
```

→ deployment 검증 — 어느 commit 이 어디 배포됐는지.

### 3.3 metrics — 메트릭 list / 조회

```bash
curl http://localhost:8080/actuator/metrics
{
  "names": [
    "jvm.memory.used", "jvm.gc.pause", "system.cpu.usage",
    "http.server.requests", "hikaricp.connections.active",
    "tomcat.threads.current", ...
  ]
}

curl http://localhost:8080/actuator/metrics/jvm.memory.used
{
  "name": "jvm.memory.used",
  "description": "...",
  "baseUnit": "bytes",
  "measurements": [{ "statistic": "VALUE", "value": 134217728 }],
  "availableTags": [
    { "tag": "area", "values": ["nonheap", "heap"] },
    { "tag": "id",   "values": ["G1 Eden Space", "G1 Old Gen", ...] }
  ]
}

# tag 필터링
curl "http://localhost:8080/actuator/metrics/jvm.memory.used?tag=area:heap"
```

→ 개별 메트릭 확인 용이. Prometheus 가 scrape 하는 것은 다음 endpoint.

### 3.4 prometheus — Prometheus 포맷 export

```bash
curl http://localhost:8080/actuator/prometheus
# HELP jvm_memory_used_bytes The amount of used memory
# TYPE jvm_memory_used_bytes gauge
jvm_memory_used_bytes{area="heap",id="G1 Eden Space",} 1.342177E8
jvm_memory_used_bytes{area="heap",id="G1 Old Gen",} 6.7108864E7
...

# HELP http_server_requests_seconds
# TYPE http_server_requests_seconds histogram
http_server_requests_seconds_bucket{method="POST",status="200",uri="/restaurants/{id}/waitings",le="0.005",} 1234.0
http_server_requests_seconds_bucket{...,le="0.01",} 2456.0
http_server_requests_seconds_count{...} 5678.0
http_server_requests_seconds_sum{...} 12.345
```

→ Prometheus 가 이 endpoint 를 주기적으로 scrape. 자세한 PromQL / 대시보드는 [[prometheus-grafana]].

### 3.5 기타 자주 쓰는 endpoint

| endpoint | 사용 |
| --- | --- |
| `/actuator/loggers` | Runtime Level 조회 / 변경 ([[../logging/fundamentals]] §4.2) |
| `/actuator/env` | 환경변수 / config |
| `/actuator/configprops` | `@ConfigurationProperties` 의 현재 값 |
| `/actuator/beans` | Spring Bean 목록 |
| `/actuator/mappings` | URL → handler 매핑 |
| `/actuator/threaddump` | thread dump (deadlock 진단) |
| `/actuator/heapdump` | heap dump 다운로드 (.hprof) — 분석에 jvisualvm / Eclipse MAT |
| `/actuator/scheduledtasks` | `@Scheduled` 작업 |
| `/actuator/httpexchanges` | 최근 HTTP request (기본 100) |
| `/actuator/shutdown` | graceful 종료 (POST) — 기본 비활성 |

---

## 4. Micrometer — 메트릭 라이브러리

Spring Boot Actuator 의 메트릭 백엔드. 다양한 monitoring 시스템 (Prometheus / Datadog / CloudWatch / New Relic / ...) 으로 export 가능.

### 4.1 4 instrument

| instrument | 정의 | 예 |
| --- | --- | --- |
| **Counter** | 단조 증가 (감소 X) | request 수 / error 수 |
| **Gauge** | 임의 값 (오르락 내리락) | queue 크기 / 활성 connection 수 |
| **Timer** | 짧은 이벤트의 시간 + 횟수 + 분포 | HTTP latency / DB query 시간 |
| **DistributionSummary** | 일반 분포 (시간이 아닌 값) | request payload 크기 / batch size |

### 4.2 Counter

```java
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Counter;

@Service
public class RestaurantService {
    private final Counter waitingCreated;

    public RestaurantService(MeterRegistry registry) {
        this.waitingCreated = Counter.builder("restaurant.waiting.created")
            .description("Number of waitings created")
            .tag("type", "online")
            .register(registry);
    }

    public void createWaiting(...) {
        // ...
        waitingCreated.increment();
        // 또는 waitingCreated.increment(5);
    }
}
```

→ `restaurant_waiting_created_total{type="online"}` 으로 Prometheus 에 노출.

### 4.3 Gauge

```java
@Component
public class QueueGauge {
    private final ConcurrentLinkedQueue<Task> queue = ...;

    public QueueGauge(MeterRegistry registry) {
        Gauge.builder("task.queue.size", queue, ConcurrentLinkedQueue::size)
            .description("Current task queue size")
            .register(registry);
    }
}
```

→ Prometheus scrape 시점에 `queue::size` 호출.

### 4.4 Timer

```java
@Service
public class PaymentService {
    private final Timer paymentTimer;

    public PaymentService(MeterRegistry registry) {
        this.paymentTimer = Timer.builder("payment.process")
            .description("Payment processing time")
            .publishPercentiles(0.5, 0.95, 0.99)
            .publishPercentileHistogram()
            .register(registry);
    }

    public PaymentResult process(Payment payment) {
        return paymentTimer.record(() -> {
            // 실제 처리
            return externalApi.charge(payment);
        });
    }
}
```

→ Prometheus 에 `payment_process_seconds_count`, `_sum`, `_bucket` 노출. p95/p99 자동 계산.

### 4.5 `@Timed` annotation

```java
@Timed(value = "payment.process", percentiles = {0.5, 0.95, 0.99})
public PaymentResult process(Payment payment) {
    // 자동 timing
}
```

→ AspectJ 자동 적용. `TimedAspect` Bean 필요.

```java
@Bean
public TimedAspect timedAspect(MeterRegistry registry) {
    return new TimedAspect(registry);
}
```

### 4.6 DistributionSummary

```java
DistributionSummary payloadSize = DistributionSummary
    .builder("http.request.payload.size")
    .baseUnit("bytes")
    .publishPercentiles(0.5, 0.95)
    .register(registry);

payloadSize.record(request.getContentLength());
```

---

## 5. Spring 자동 측정 — 거의 모든 것

Actuator 활성화 만으로 자동 노출되는 메트릭.

### 5.1 HTTP — `http.server.requests`

```
http_server_requests_seconds_count{method="POST",status="200",uri="/restaurants/{id}/waitings"}  5678
http_server_requests_seconds_sum{...}  123.456
http_server_requests_seconds_bucket{le="0.005",...}  1234
http_server_requests_seconds_max{...}  0.789
```

| tag | 의미 |
| --- | --- |
| method | GET / POST / PUT / ... |
| status | 200 / 201 / 400 / 500 |
| uri | URL pattern (`/restaurants/{id}` — 실제 path variable 치환) |
| exception | 예외 발생 시 class name |
| outcome | SUCCESS / CLIENT_ERROR / SERVER_ERROR |

→ p95 latency / RPS / 에러율을 Prometheus 만으로 즉시 산출.

### 5.2 JVM

| 메트릭 | 의미 |
| --- | --- |
| `jvm.memory.used` | heap / non-heap 메모리 사용량 |
| `jvm.memory.max` | 최대치 |
| `jvm.gc.pause` | GC pause 시간 |
| `jvm.gc.live.data.size` | live heap 크기 |
| `jvm.threads.live` | 현재 thread 수 |
| `jvm.threads.daemon` | daemon thread 수 |
| `jvm.classes.loaded` | 로드된 class 수 |
| `jvm.buffer.memory.used` | NIO buffer |

### 5.3 시스템

| 메트릭 | 의미 |
| --- | --- |
| `system.cpu.usage` | host CPU 사용률 (0-1) |
| `process.cpu.usage` | 본 process CPU 사용률 |
| `system.load.average.1m` | load average |
| `disk.free` / `disk.total` | 디스크 |
| `process.uptime` | uptime 초 |

### 5.4 Hikari (DB connection pool)

```
hikaricp_connections_active     5      # 사용 중
hikaricp_connections_idle       15     # 대기
hikaricp_connections_max        20     # 한도
hikaricp_connections_pending    0      # connection 대기 중인 thread
hikaricp_connections_acquire_seconds   # connection 획득 시간
```

→ pending > 0 이 지속되면 pool 부족 — `spring.datasource.hikari.maximum-pool-size` 증가 검토.

### 5.5 Tomcat

| 메트릭 | 의미 |
| --- | --- |
| `tomcat.threads.current` | 현재 worker thread |
| `tomcat.threads.busy` | 처리 중 |
| `tomcat.threads.config.max` | 최대 |
| `tomcat.sessions.active.current` | 활성 session |
| `tomcat.global.received` / `tomcat.global.sent` | bytes |

→ busy 가 max 에 근접하면 thread 부족.

### 5.6 기타 자동 측정

- Spring Cache — `cache.gets`, `cache.puts`, `cache.evictions`
- Spring Data Repository — `spring.data.repository.invocations`
- HTTP Client (RestTemplate / WebClient) — `http.client.requests`
- Logback - `logback.events` (level 별 카운트)
- Scheduled tasks — `spring.scheduled.tasks`

---

## 6. 메트릭 작명 / tag 규칙

### 6.1 작명

- `dot.separated.lower.case` — Micrometer 표준 (`restaurant.waiting.created`)
- Prometheus 로 export 시 자동 `_` 변환 (`restaurant_waiting_created`)
- Counter 는 `_total` suffix 자동 추가

### 6.2 tag 의 cardinality

| 좋음 | 나쁨 |
| --- | --- |
| `status="200"` (가능 값 ~10) | `user_id="user_789"` (가능 값 수만 ~수억) |
| `method="POST"` | `request_id="a7c3..."` |
| `uri="/restaurants/{id}"` (path variable 치환) | `uri="/restaurants/12345"` (실제 ID) |

→ tag 의 unique 조합이 폭주하면 Prometheus 메모리 폭주. **cardinality 가 큰 값은 tag 금지**.

### 6.3 URL tag — Spring 의 자동 path variable 치환

Spring 의 `http.server.requests` 는 `@RequestMapping("/restaurants/{id}/waitings")` 의 `{id}` 를 그대로 유지. 즉 `uri="/restaurants/{id}/waitings"` (tag 의 안전한 cardinality).

만약 직접 `Counter` 만들 때 user id 같은 high cardinality 사용하면 Prometheus 가 OOM.

---

## 7. 사용자 정의 — Custom Health Check

```java
@Component
public class ExternalApiHealthIndicator implements HealthIndicator {

    private final RestTemplate restTemplate;

    @Override
    public Health health() {
        try {
            ResponseEntity<String> response = restTemplate.getForEntity(
                "https://api.external.com/health", String.class);
            if (response.getStatusCode().is2xxSuccessful()) {
                return Health.up()
                    .withDetail("statusCode", response.getStatusCodeValue())
                    .build();
            }
            return Health.down()
                .withDetail("statusCode", response.getStatusCodeValue())
                .build();
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}
```

→ `/actuator/health` 에 자동 통합.

---

## 8. Spring Boot 의 graceful shutdown

```yaml
server:
  shutdown: graceful           # default = immediate

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

→ SIGTERM 수신 시 30초 동안 in-flight request 완료 대기. k8s / docker stop 의 표준.

---

## 9. 보안 — Production endpoint 보호

### 9.1 별도 port (management)

```yaml
management:
  server:
    port: 8081
    address: 127.0.0.1
```

→ Nginx / LB 는 8080 만 노출. 8081 은 internal monitoring 만.

### 9.2 endpoint 별 노출 제어

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
        # loggers / env / heapdump 는 production 에서 명시 제외
```

### 9.3 인증

```java
@Configuration
public class ActuatorSecurityConfig {

    @Bean
    public SecurityFilterChain actuatorSecurity(HttpSecurity http) throws Exception {
        http
            .securityMatcher(EndpointRequest.toAnyEndpoint())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(EndpointRequest.to("health", "info", "prometheus")).permitAll()
                .anyRequest().hasRole("ACTUATOR_ADMIN"))
            .httpBasic(Customizer.withDefaults());
        return http.build();
    }
}
```

→ health / info / prometheus 는 공개 (k8s / Prometheus 가 호출). 나머지는 인증 필수.

---

## 10. k8s 통합

### 10.1 liveness / readiness probe

```yaml
# k8s deployment
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8081
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8081
  initialDelaySeconds: 10
  periodSeconds: 5
```

### 10.2 PodMonitor (Prometheus Operator)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: restaurant-api
spec:
  selector:
    matchLabels: { app: restaurant-api }
  podMetricsEndpoints:
    - port: management
      path: /actuator/prometheus
      interval: 15s
```

상세: [[prometheus-grafana]] §4.

---

## 11. 학습 / 적용 절차

| 단계 | 작업 |
| --- | --- |
| 1 | `spring-boot-starter-actuator` 의존성 추가 |
| 2 | `application.yml` 에 endpoint 노출 (health / info / metrics / prometheus) |
| 3 | `/actuator/health` 검증 |
| 4 | `micrometer-registry-prometheus` 의존성 추가 |
| 5 | `/actuator/prometheus` 검증 |
| 6 | Custom Counter / Timer 1-2 개 추가 |
| 7 | `management.server.port=8081` 분리 |
| 8 | production 의 보안 설정 (인증 / 별도 port) |
| 9 | Prometheus 가 scrape 하도록 — [[prometheus-grafana]] |
| 10 | Grafana 대시보드 — [[prometheus-grafana]] |

---

## 12. 흔한 실패 모드

| 실패 | 원인 | 정정 |
| --- | --- | --- |
| `/actuator/prometheus` 404 | `micrometer-registry-prometheus` 의존성 없음 | 추가 |
| `/actuator/metrics` 만 보이고 `prometheus` 안 보임 | `management.endpoints.web.exposure.include` 누락 | prometheus 포함 |
| Custom Counter 가 Prometheus 에 안 보임 | scrape 시점에 instance 미생성 | 생성자에서 register, 사용 전에 |
| Prometheus 메모리 폭주 | 너무 높은 tag cardinality (user_id 등) | tag 검토 — high cardinality 제거 |
| health 가 "UP" 인데 DB 죽음 | DB indicator 활성 안 됨 | spring-boot-starter-data-jpa + spring-boot-starter-jdbc + DataSource bean 확인 |
| heap dump 가 외부에서 다운로드됨 | `/actuator/heapdump` 노출 | production 노출 제거 |
| logger Level 외부에서 변경됨 | `/actuator/loggers` POST 인증 없음 | 인증 + 내부 망 만 |
| k8s readiness 가 true 인데 트래픽 받자 5xx | 초기화 미완료 | `ApplicationAvailability` API 로 readyState 제어 |
| graceful shutdown 안 됨 | `server.shutdown=graceful` 누락 | 추가 |
| management port 가 main port 와 같음 | production 노출 위험 | `management.server.port=8081` 분리 |

---

## 13. 참고

- [[monitoring|↑ monitoring]]
- [[prometheus-grafana]]
- [[../logging/elk-stack]]
- [[../observability]]
- Spring Boot Actuator 공식 docs — https://docs.spring.io/spring-boot/reference/actuator/
- Micrometer 공식 docs — https://docs.micrometer.io
- 강의: §7 (Actuator / Micrometer 로 서버 모니터링)
