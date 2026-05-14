---
title: "Observability — Logs / Metrics / Tracing / Alerts"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T06:25:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - operations
  - observability
  - logging
  - metrics
---

# Observability — Logs / Metrics / Tracing / Alerts

**[[operations|↑ operations hub]]**

> "운영 중 무슨 일이 일어나는지 보이나" — 부실하면 **사고 발생 후 원인 추적 X, 의심 활동 감지 X**.

---

## 1. 3 pillars of observability

| Pillar | 무엇 | 도구 |
| --- | --- | --- |
| **Logs** | 이벤트 history | CloudWatch / Datadog / ELK |
| **Metrics** | 시간 별 숫자 (counter / gauge / histogram) | Prometheus / Micrometer / Datadog |
| **Traces** | 분산 호출 추적 | Jaeger / X-Ray / Datadog |

---

## 2. Logging

### 2.1 로그 레벨 정책

```yaml
# application-prod.yml
logging:
  level:
    root: INFO
    com.example: INFO
    org.springframework.security: INFO
    org.hibernate.SQL: WARN                  # 운영 DEBUG 금지
    org.hibernate.orm.jdbc.bind: WARN
    org.springframework.transaction: INFO
```

**왜 prod = INFO+**
- DEBUG = SQL parameter (password / phone 평문) 노출.
- INFO = 비즈니스 이벤트만.

**dev / staging**
- DEBUG OK (디버깅 필요).
- 단 prod 와 다른 DB.

### 2.2 Structured Logging

```kotlin
implementation("net.logstash.logback:logstash-logback-encoder:7.4")
```

```xml
<!-- logback-spring.xml -->
<appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <includeMdcKeyName>traceId</includeMdcKeyName>
        <includeMdcKeyName>spanId</includeMdcKeyName>
        <includeMdcKeyName>userId</includeMdcKeyName>
        <includeMdcKeyName>requestId</includeMdcKeyName>
        <customFields>{"app":"yule-studio","env":"prod"}</customFields>
    </encoder>
</appender>
```

**왜 JSON**
- 검색 / aggregation (Athena / Kibana).
- text 로그는 grep 만 가능.

### 2.3 MDC (Mapped Diagnostic Context)

```java
@Component
public class TraceIdFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
        String traceId = ((HttpServletRequest) req).getHeader("X-Request-Id");
        if (traceId == null) traceId = UlidCreator.getMonotonicUlid().toString();

        try (var ignored = MDC.putCloseable("traceId", traceId)) {
            chain.doFilter(req, res);
        }
    }
}
```

**왜 MDC**
- 같은 요청의 여러 로그 자동 연결.
- 분산 추적의 핵심.

### 2.4 마스킹 정책

자세히: [[../security/sensitive-data-handling]].

**요약**
- password 평문 — 절대 X.
- email / phone — 마스킹.
- JWT — 로그 X.
- user_id (ULID) — OK.

---

## 3. Metrics — Prometheus / Micrometer

### 3.1 본 vault 의 핵심 메트릭

| 메트릭 | 타입 | 의미 |
| --- | --- | --- |
| `signup.attempts.total{result="success"}` | Counter | 성공 가입 수 |
| `signup.attempts.total{result="duplicate"}` | Counter | 중복 이메일 (enumeration spike 감지) |
| `signup.attempts.total{result="weak_password"}` | Counter | 약한 비번 |
| `signup.attempts.total{result="error"}` | Counter | 시스템 에러 |
| `signup.latency` | Timer | p50/p95/p99 응답 시간 |
| `login.attempts.total{result="success/failed"}` | Counter | 로그인 |
| `login.failure.rate` | Gauge | 분당 실패 rate (brute force 감지) |
| `password.hash.duration` | Timer | argon2 비용 (튜닝 지표) |
| `email_outbox.pending` | Gauge | 큐 길이 (워커 이상 감지) |
| `email_outbox.sent.rate` | Counter | 발송 rate |
| `db.connection.pool.active` | Gauge | DB connection 활용도 |
| `jvm.memory.used` | Gauge | 메모리 사용 |

### 3.2 코드

```java
@Service
@RequiredArgsConstructor
public class SignupUseCase {
    private final MeterRegistry meter;

    @Transactional
    public User execute(SignupCommand cmd) {
        var sample = Timer.start(meter);
        try {
            var user = doExecute(cmd);
            meter.counter("signup.attempts.total", "result", "success").increment();
            return user;
        } catch (EmailAlreadyExistsException e) {
            meter.counter("signup.attempts.total", "result", "duplicate").increment();
            throw e;
        } catch (BusinessException e) {
            meter.counter("signup.attempts.total", "result", e.getResponseCode().code()).increment();
            throw e;
        } catch (Exception e) {
            meter.counter("signup.attempts.total", "result", "error").increment();
            throw e;
        } finally {
            sample.stop(meter.timer("signup.latency"));
        }
    }
}
```

### 3.3 endpoint 활성

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true
        signup.latency: true
```

→ `/actuator/prometheus` 가 Prometheus scrape endpoint.

---

## 4. Tracing — OpenTelemetry / X-Ray

### 4.1 왜 필요

- MSA / 다중 서비스 호출 시 — 어디서 느린지 추적.
- 모놀리식도 — DB / external API 호출 별 latency 분리.

### 4.2 통합

```kotlin
implementation("org.springframework.boot:spring-boot-starter-actuator")
implementation("io.micrometer:micrometer-tracing-bridge-otel")
implementation("io.opentelemetry:opentelemetry-exporter-otlp")
```

### 4.3 활성

```yaml
management:
  tracing:
    sampling:
      probability: 0.1                  # 10% 만 (비용 절감)
  otlp:
    tracing:
      endpoint: http://collector:4318/v1/traces
```

**왜 10% sampling**
- 100% = 비용 + 부담 ↑.
- 10% = 통계적 유의 + 비용 ↓.
- 에러 발생 시 100% (sampling-on-error).

---

## 5. Alerts — 알람 정책

### 5.1 알람 채널

| 우선순위 | 채널 | 사용 |
| --- | --- | --- |
| **P1** | PagerDuty (즉시 호출) | 5xx 폭증 / DB down |
| **P2** | Slack #alerts | 운영 의심 |
| **P3** | Slack #monitoring | 정보 (모니터링) |

### 5.2 본 vault 알람

| 시그널 | 임계 | 우선 |
| --- | --- | --- |
| 5xx 비율 > 5% (5분) | PagerDuty | P1 |
| Login failure rate > 20% | Slack #alerts | P2 (brute force 의심) |
| Login failure 같은 IP > 100/min | Slack #security | P2 |
| Signup duplicate rate > 30% | Slack #security | P2 (enumeration 시도) |
| Email outbox pending > 1000 | Slack #alerts | P2 (워커 이상) |
| Email outbox failure rate > 10% | Slack #alerts | P2 (SES 문제) |
| DB connection pool > 80% | PagerDuty | P1 |
| JVM heap > 90% | PagerDuty | P1 |
| password.hash.duration p95 > 1s | Slack | P3 |
| `auth_audit_log` SUSPICIOUS_ACTIVITY > 5/hour | Slack #security | P2 |

### 5.3 Grafana / Datadog alert 예시

```yaml
- alert: SignupFailureSpike
  expr: |
    rate(signup_attempts_total{result!="success"}[5m])
      / rate(signup_attempts_total[5m]) > 0.3
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Signup failure rate > 30% — possible enumeration or system issue"
```

---

## 6. Dashboard — Grafana

### 6.1 본 vault 의 표준 대시보드

```
┌────────────────────────────────────────┐
│ Auth Overview                          │
├────────────────────────────────────────┤
│ Signup rate (/min) | Login rate        │
│ Success rate %     | Failure rate %    │
│ p95 latency        | p99 latency       │
├────────────────────────────────────────┤
│ Detailed metrics                       │
│  - Signup by result                    │
│  - Login by result                     │
│  - Email outbox queue                  │
│  - Token refresh rate                  │
├────────────────────────────────────────┤
│ Security                                │
│  - Failed login by IP (top 10)         │
│  - Reuse detection events              │
│  - Account lock count                  │
├────────────────────────────────────────┤
│ Infra                                   │
│  - DB connection pool                  │
│  - JVM heap                            │
│  - Network latency                     │
└────────────────────────────────────────┘
```

---

## 7. Error Tracking — Sentry

### 7.1 통합

```kotlin
implementation("io.sentry:sentry-spring-boot-starter-jakarta:7.14.0")
```

```yaml
sentry:
  dsn: https://...@sentry.io/...
  environment: ${spring.profiles.active}
  release: ${app.version}
  send-default-pii: false               # PII 자동 수집 X
  traces-sample-rate: 0.1
```

### 7.2 PII Scrubbing

```yaml
sentry:
  send-default-pii: false
  in-app-includes:
    - com.example.shop
```

```java
Sentry.configureScope(scope -> {
    scope.setUser(new User(maskEmail(user.email().value()), user.id().value()));
});
```

**왜 PII scrubbing**
- Sentry 가 자동으로 request body / cookie 수집.
- 평문 password / email 등 노출 위험.

---

## 8. 함정 모음

### 함정 1 — Hibernate SQL log 운영 ON
SQL parameter (password / phone) 평문 노출.
→ prod = WARN.

### 함정 2 — 로그에 평문 password / JWT
git / log aggregator 유출.
→ 마스킹 + filter.

### 함정 3 — 메트릭 없음
사고 발생 후 분석 X.
→ 핵심 메트릭 + dashboard.

### 함정 4 — 알람 너무 많음 (noise)
실제 사고 시 묻힘.
→ 우선순위 + 정기 review.

### 함정 5 — Trace sampling 100%
비용 폭증.
→ 10% + on-error.

### 함정 6 — MDC 안 전파 (async 환경)
@Async 안에서 traceId 없음.
→ TaskDecorator 로 전파.

### 함정 7 — Counter 이름 inconsistency
`signup.attempts.total` vs `signup_attempts_total` 혼용.
→ Naming 가이드 (Micrometer convention).

### 함정 8 — Tag 폭증 (cardinality 폭망)
`tag("userId", ...)` — 1억 user = 1억 cardinality.
→ Counter 의 label 은 low-cardinality (status, result).

### 함정 9 — Sentry 의 PII scrubbing 비활성
breadcrumb / context 가 자동 PII 수집.
→ send-default-pii: false + 명시 scrubbing.

### 함정 10 — Log retention 너무 짧음 / 길음
짧음 (1주) = 사고 분석 X.
길음 (5년) = 비용 폭증.
→ CloudWatch 30일 + S3 1년+.

### 함정 11 — Structured logging 없이 grep 의존
검색 / aggregation 어려움.
→ JSON 형식 + Kibana / Datadog.

### 함정 12 — Alert 의 acknowledge 없음
같은 알람 반복.
→ PagerDuty escalation + acknowledge 정책.

---

## 9. 관련

- [[operations|↑ operations hub]]
- [[../security/audit-logging]] — 보안 audit
- [[deployment]] — 배포 후 모니터링
- [[runbook]] — 알람 시 대응
- 외부 — Micrometer Docs, Prometheus Best Practices, OpenTelemetry
