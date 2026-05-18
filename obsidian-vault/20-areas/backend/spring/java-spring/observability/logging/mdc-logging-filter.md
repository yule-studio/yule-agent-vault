---
title: "logging/mdc-logging-filter — request_id 부여 + Servlet Filter"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-19T06:30:00+09:00
tags: [backend, java-spring, observability, logging, mdc, filter]
home_hub: logging
related:
  - "[[logging]]"
  - "[[fundamentals]]"
  - "[[logback-configuration]]"
  - "[[elk-stack]]"
---

# logging/mdc-logging-filter — request_id 부여 + Servlet Filter

**[[logging|↑ logging]]** · **[[../observability|↑ observability]]**

> 강의 매핑: §3.10-11 (로그 읽기 + MDC Logging Filter with UUID).

---

## 1. 목적

본 문서는 Spring Boot 응용의 각 HTTP request 에 고유 식별자 (`request_id`) 를 부여해 모든 로그에서 추적 가능하게 하는 **MDC (Mapped Diagnostic Context) + Servlet Filter** 패턴을 정의한다.

본 문서가 정의하는 것:
- MDC 의 정의 + 작동 원리 (ThreadLocal)
- Servlet Filter 로 request_id 부여
- 외부에서 받은 trace id 전파 (`X-Request-Id` / `X-B3-TraceId`)
- 비동기 / `@Async` / `CompletableFuture` 경계에서 MDC 보존
- Logback pattern 의 `%X{key}`
- 다중 key 사용 (user_id / tenant_id / request_id 같이)

본 문서가 정의하지 않는 것:
- 로그의 일반 사용 (Level / 구성) — [[fundamentals]]
- Logback 의 파일 / 회전 — [[logback-configuration]]
- OpenTelemetry / Jaeger 의 trace propagation — 별도

---

## 2. 왜 MDC 인가

### 2.1 문제

```
2026-05-19 10:23:11 INFO  RestaurantController - 식당 12345 웨이팅 등록 시작
2026-05-19 10:23:11 INFO  RestaurantService - 빈자리 검사
2026-05-19 10:23:11 INFO  PaymentService - 결제 검증
2026-05-19 10:23:11 INFO  RestaurantController - 식당 12345 웨이팅 등록 시작     ← 다른 user
2026-05-19 10:23:11 ERROR PaymentService - 카드 거절
```

→ 어느 request 가 ERROR 인지 식별 불가. 동시 traffic 100 req/s 면 추적 불가능.

### 2.2 해결

```
2026-05-19 10:23:11 INFO  RestaurantController [req=a7c3] - 식당 12345 웨이팅 등록 시작
2026-05-19 10:23:11 INFO  RestaurantService   [req=a7c3] - 빈자리 검사
2026-05-19 10:23:11 INFO  PaymentService      [req=a7c3] - 결제 검증
2026-05-19 10:23:11 INFO  RestaurantController [req=b9f1] - 식당 12345 웨이팅 등록 시작
2026-05-19 10:23:11 ERROR PaymentService      [req=b9f1] - 카드 거절
```

→ `grep req=b9f1` 으로 한 request 의 모든 로그 즉시 추출.

---

## 3. MDC 의 정의

| 항목 | 의미 |
| --- | --- |
| **MDC** | SLF4J 의 `Mapped Diagnostic Context`. 현재 thread 에 key-value 를 저장. |
| 저장소 | `ThreadLocal<Map<String, String>>` |
| 자동 출력 | Logback pattern 의 `%X{key}` 가 자동 inject |
| 정리 책임 | request 끝나면 명시적 `MDC.clear()` 필수 (Tomcat thread 재사용) |

### 3.1 기본 사용

```java
import org.slf4j.MDC;

MDC.put("request_id", "a7c3...");
log.info("식당 웨이팅 등록 시작");        // pattern 의 %X{request_id} 가 자동 출력
MDC.remove("request_id");                  // 또는 MDC.clear()
```

### 3.2 Logback pattern

```xml
<encoder>
  <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{40} [%X{request_id:-}] - %msg%n</pattern>
</encoder>
```

→ `:-` 는 MDC 에 없을 때의 기본값 (빈 문자열).

---

## 4. Servlet Filter — request 마다 request_id 부여

### 4.1 표준 구현

```java
package com.example.common.logging;

import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.slf4j.MDC;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.UUID;

@Component
@Order(Ordered.HIGHEST_PRECEDENCE)               // 가장 먼저 실행
public class MdcLoggingFilter implements Filter {

    public static final String REQUEST_ID  = "request_id";
    public static final String HEADER_NAME = "X-Request-Id";

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {

        HttpServletRequest  httpReq = (HttpServletRequest)  req;
        HttpServletResponse httpRes = (HttpServletResponse) res;

        // 1. 외부에서 받은 id 가 있으면 전파, 없으면 새로 생성
        String requestId = httpReq.getHeader(HEADER_NAME);
        if (requestId == null || requestId.isBlank()) {
            requestId = UUID.randomUUID().toString().substring(0, 8);
        }

        try {
            // 2. MDC 에 저장
            MDC.put(REQUEST_ID, requestId);

            // 3. response header 에 echo — 클라이언트 / proxy 가 추적 가능
            httpRes.setHeader(HEADER_NAME, requestId);

            // 4. 다음 filter 진행
            chain.doFilter(req, res);

        } finally {
            // 5. ★ 반드시 clear — Tomcat thread pool 재사용 시 leak 방지
            MDC.clear();
        }
    }
}
```

### 4.2 `@Component` vs `FilterRegistrationBean`

| 방법 | 동작 |
| --- | --- |
| `@Component + Filter implements` | Spring Boot 가 자동 등록 |
| `FilterRegistrationBean` Bean 정의 | URL pattern / Order / async 세밀 제어 |

```java
@Bean
public FilterRegistrationBean<MdcLoggingFilter> mdcLoggingFilter() {
    FilterRegistrationBean<MdcLoggingFilter> bean = new FilterRegistrationBean<>();
    bean.setFilter(new MdcLoggingFilter());
    bean.setOrder(Ordered.HIGHEST_PRECEDENCE);
    bean.addUrlPatterns("/*");
    bean.setAsyncSupported(true);
    return bean;
}
```

### 4.3 결과

```
$ curl -i http://localhost:8080/restaurants/12345/waitings -X POST
HTTP/1.1 201
X-Request-Id: a7c3b421
Content-Type: application/json
...

$ grep "request_id=a7c3b421" app.log    또는    grep "[a7c3b421]" app.log
2026-05-19 10:23:11.142 [http-nio-8080-exec-3] INFO  c.e.r.RestaurantController [a7c3b421] - 식당 12345 웨이팅 등록 시작
2026-05-19 10:23:11.153 [http-nio-8080-exec-3] INFO  c.e.r.RestaurantService   [a7c3b421] - 빈자리 검사
2026-05-19 10:23:11.187 [http-nio-8080-exec-3] INFO  c.e.r.RestaurantController [a7c3b421] - 식당 12345 웨이팅 등록 완료 - user=user_789
```

---

## 5. 외부 trace id 전파 (gateway / proxy / 다중 서비스)

### 5.1 표준 header

| header | 의미 |
| --- | --- |
| `X-Request-Id` | 일반 (de facto 표준) |
| `X-Correlation-Id` | 다중 서비스 추적 |
| `X-B3-TraceId` | Zipkin / B3 propagation (구) |
| `traceparent` | W3C Trace Context (현행 표준) |

→ 프록시 / API Gateway (nginx / AWS ALB / Cloudflare) 가 header 추가 가능. 받은 값이 있으면 그대로, 없으면 생성.

### 5.2 W3C traceparent 형식

```
traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01
              │  └────── trace id (32 hex) ──────┘ └─ span id (16) ─┘ │
              version                                                  flags
```

→ MDC 에 `trace_id`, `span_id` 분리 저장. Logstash / Elasticsearch / Grafana Tempo 에서 연결.

### 5.3 외부 → 내부 → 외부 전파 예

```java
String traceparent = httpReq.getHeader("traceparent");
String traceId = (traceparent != null && traceparent.length() >= 35)
                 ? traceparent.substring(3, 35)
                 : generateNewTraceId();
MDC.put("trace_id", traceId);

// 외부 호출 시 같은 traceparent 전달
HttpHeaders headers = new HttpHeaders();
headers.set("traceparent", traceparent);
restTemplate.exchange(url, GET, new HttpEntity<>(headers), ...);
```

---

## 6. 다중 key — user_id / tenant_id / request_id 같이

```java
// 인증 Filter 또는 Interceptor 가 user 정보 알게 되는 시점
MDC.put("user_id", currentUser.id());
MDC.put("tenant_id", currentUser.tenantId());
```

```xml
<!-- logback pattern -->
<pattern>%d{HH:mm:ss.SSS} [%X{request_id:-}] [%X{user_id:-}] [%X{tenant_id:-}] %-5level %logger{40} - %msg%n</pattern>
```

→ Kibana 의 검색 `user_id:"user_789"` 로 한 사용자의 모든 활동 추적.

---

## 7. 비동기 / @Async / CompletableFuture 의 MDC

### 7.1 문제

MDC 는 `ThreadLocal` → 새 thread 로 작업 위임하면 사라짐.

```java
@Async
public CompletableFuture<Result> processAsync() {
    // 여기서 MDC.get("request_id") = null
    log.info("처리 중");    // [request_id=] 빈 값
}
```

### 7.2 해결 — `MdcAsyncTaskExecutor`

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setTaskDecorator(new MdcTaskDecorator());     // ★
        executor.initialize();
        return executor;
    }

    static class MdcTaskDecorator implements TaskDecorator {
        @Override
        public Runnable decorate(Runnable runnable) {
            Map<String, String> mdc = MDC.getCopyOfContextMap();
            return () -> {
                try {
                    if (mdc != null) MDC.setContextMap(mdc);
                    runnable.run();
                } finally {
                    MDC.clear();
                }
            };
        }
    }
}
```

### 7.3 `CompletableFuture.supplyAsync` 도 같은 방식

```java
CompletableFuture.supplyAsync(() -> { ... }, taskExecutor);    // 위 executor 사용
```

### 7.4 Reactor (WebFlux)

Reactor 는 thread 가 자유롭게 바뀜 → MDC 가 그대로 안 됨. `Context` 또는 Micrometer Context Propagation 사용.

```java
// Mono / Flux 의 context
Mono.deferContextual(ctx -> {
    String requestId = ctx.get("request_id");
    // ...
    return Mono.just(...);
});
```

상세는 별도 (WebFlux 영역).

---

## 8. AccessLog vs ApplicationLog

| 종류 | 정의 | 위치 |
| --- | --- | --- |
| **AccessLog** | 각 HTTP request 의 진입 / 응답 (한 줄) | 별도 file appender |
| **ApplicationLog** | 비즈니스 로직의 로그 | 기본 appender |

### 8.1 Spring 의 AccessLog Filter (간단)

```java
@Component
@Order(Ordered.LOWEST_PRECEDENCE)
public class AccessLogFilter implements Filter {

    private static final Logger access = LoggerFactory.getLogger("ACCESS");

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {

        long start = System.currentTimeMillis();
        HttpServletRequest  httpReq = (HttpServletRequest)  req;
        HttpServletResponse httpRes = (HttpServletResponse) res;

        try {
            chain.doFilter(req, res);
        } finally {
            long elapsed = System.currentTimeMillis() - start;
            access.info("{} {} {} {}ms",
                httpReq.getMethod(),
                httpReq.getRequestURI(),
                httpRes.getStatus(),
                elapsed);
        }
    }
}
```

→ Logback 에서 `ACCESS` logger 만 별도 file 로 분리.

---

## 9. 보안 — request_id 의 노출

| 위치 | 노출 적정 여부 |
| --- | --- |
| Application 로그 / Kibana | ✓ (내부) |
| Response Header `X-Request-Id` | ✓ (CS 대응 용이) |
| 사용자 응답 body | ✓ (error response 의 `traceId` 필드 권장) |
| 외부 호출 header | ✓ (W3C traceparent 표준) |

→ request_id 는 **민감 데이터 아님**. CS 가 사용자에게 "request id 알려 주세요" 라고 요청 가능.

---

## 10. 흔한 실패 모드

| 실패 | 원인 | 정정 |
| --- | --- | --- |
| MDC value 가 다른 request 에 보임 | finally MDC.clear() 누락 | 항상 `try-finally` 로 clear |
| `@Async` 에서 MDC 사라짐 | ThreadLocal — 새 thread | MdcTaskDecorator 적용 |
| `%X{request_id}` 가 항상 `null` | MDC.put 이전에 logger 호출 / pattern 오타 | filter @Order(HIGHEST) + pattern 검증 |
| 외부에서 받은 id 가 무시됨 | `getHeader` 누락 | header 우선 → fallback UUID |
| 한 request 의 로그가 여러 thread 에 흩어짐 (정상) | async / parallel — 정상 동작 | MDC 전파만 보장 |
| UUID 너무 김 (36자) | gateway 자동 생성 짧은 ID | `substring(0, 8)` 또는 nanoid |
| Logstash 의 JSON 에 MDC 안 보임 | LogstashEncoder customField 미설정 | `<provider class="...MdcJsonProvider"/>` |
| 같은 request 의 응답 후에도 thread 가 다음 request 의 로그를 옛 id 로 출력 | clear 누락 + thread 재사용 | finally clear |

---

## 11. 적용 체크 항목

| 항목 | 점검 |
| --- | --- |
| `MdcLoggingFilter` 가 `@Order(HIGHEST_PRECEDENCE)` |  |
| `MDC.clear()` 가 `finally` 안 |  |
| Logback pattern 에 `%X{request_id:-}` 포함 |  |
| Response Header `X-Request-Id` 에 echo |  |
| 외부 header (X-Request-Id / traceparent) 가 있으면 그대로 사용 |  |
| `@Async` / `ThreadPoolTaskExecutor` 가 `MdcTaskDecorator` 적용 |  |
| 한 request 의 모든 로그가 같은 request_id 보유 (테스트) |  |
| `grep request_id=...` 으로 한 request 의 전체 흐름 추출 가능 |  |

---

## 12. 참고

- [[logging|↑ logging]]
- [[fundamentals]]
- [[logback-configuration]]
- [[elk-stack]]
- SLF4J MDC docs — https://www.slf4j.org/api/org/slf4j/MDC.html
- W3C Trace Context — https://www.w3.org/TR/trace-context/
- Spring Boot Filter Order — https://docs.spring.io/spring-boot/reference/web/servlet.html#web.servlet.embedded-container.servlets-filters-listeners
- Micrometer Context Propagation — https://docs.micrometer.io/context-propagation/
