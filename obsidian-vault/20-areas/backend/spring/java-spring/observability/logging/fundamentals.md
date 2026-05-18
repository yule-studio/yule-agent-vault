---
title: "logging/fundamentals — 왜 / 무엇 / Level / 8 구성요소"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-19T06:15:00+09:00
tags: [backend, java-spring, observability, logging, fundamentals]
home_hub: logging
related:
  - "[[logging]]"
  - "[[mdc-logging-filter]]"
  - "[[logback-configuration]]"
  - "[[elk-stack]]"
---

# logging/fundamentals — 왜 / 무엇 / Level / 8 구성요소

**[[logging|↑ logging]]** · **[[../observability|↑ observability]]**

> 강의 매핑: §2 (로그의 기본) + §3.7-9 (Level / 8 구성요소).

---

## 1. 목적

본 문서는 로그가 왜 필요하고, 무엇을 기록하며, 6 가지 Level 의 의미, 1 줄 로그의 8 구성요소를 정의한다.

본 문서가 정의하는 것:
- 로그가 해결하는 4 문제 (CS 대응 / 장애 분석 / 보안 audit / 비즈니스 분석)
- "무엇을 기록할까" 의 3 분류 (request / state change / 예외)
- 6 Level (TRACE / DEBUG / INFO / WARN / ERROR / FATAL) 의 정의 + 사용 기준
- 1 줄 로그의 8 구성요소
- SLF4J + Logback 의 표준 사용법
- 로그 안티패턴

본 문서가 정의하지 않는 것:
- MDC / request id — [[mdc-logging-filter]]
- Logback 의 파일 / 회전 / 압축 설정 — [[logback-configuration]]
- ELK 수집 / 검색 — [[elk-stack]]

---

## 2. 왜 로그가 필요한가

| 시점 | 로그 없으면 | 로그 있으면 |
| --- | --- | --- |
| **CS 대응** | "고객 A 의 결제가 왜 실패?" → 알 수 없음 | request id 로 5초 만에 추적 |
| **장애 분석** | 5xx 발생 후 원인 추정 | stack trace + 직전 context |
| **보안 audit** | 누가 언제 무엇을 했는지 모름 | 로그인 / 권한 변경 기록 |
| **비즈니스 분석** | 사용자 행동 추정만 | 실제 이벤트 집계 |

→ 로그는 **운영의 눈** 이다. 로그 없는 시스템 = 블랙박스.

---

## 3. "무엇을 기록할까"

### 3.1 기록 대상 — 3 분류

| 분류 | 예 | Level |
| --- | --- | --- |
| **request / response** | API 진입 / 완료 + 응답 시간 + 상태 코드 | INFO |
| **state change** | 회원가입 / 결제 완료 / 권한 변경 / 외부 호출 | INFO |
| **예외 / 비정상** | 검증 실패 / DB 에러 / timeout / 의도된 거부 | WARN / ERROR |

### 3.2 기록 금지 / 주의

| 데이터 | 처리 |
| --- | --- |
| password / API key / token | 절대 로그 X |
| 카드 번호 / 주민번호 | 마스킹 (마지막 4자리만) |
| 개인정보 (이메일 / 전화) | 환경별 정책 (개발=노출 / production=마스킹) |
| request body 전체 | 큰 데이터 / 민감 정보 → 길이만 / 필드 선택 |
| 비밀번호 변경 시 새 비밀번호 | 절대 X |

→ GDPR / 개인정보보호법 / PCI-DSS 등 컴플라이언스의 첫 단계.

### 3.3 한 줄 로그 vs 구조화 로그

```
# 한 줄 (사람 읽기 쉬움)
2026-05-19 10:23:11 INFO  c.e.r.RestaurantController - 식당 12345 의 웨이팅 등록 완료 - user=user_789 wait=15

# 구조화 (machine readable — ELK 등)
{"@timestamp":"2026-05-19T10:23:11+09:00", "level":"INFO",
 "logger":"c.e.r.RestaurantController",
 "request_id":"a7c3...",
 "user_id":"user_789", "restaurant_id":"12345", "wait_count":15,
 "message":"식당 12345 의 웨이팅 등록 완료"}
```

→ 본 영역의 표준은 **양쪽 다 출력** — 사람 읽기는 console / 파일, 머신 읽기는 Logstash 로 JSON.

---

## 4. 6 Level — TRACE / DEBUG / INFO / WARN / ERROR / FATAL

### 4.1 정의 + 사용 기준

| Level | 의미 | 사용 예 | production 출력 |
| --- | --- | --- | --- |
| **TRACE** | 가장 상세 (변수 값 / loop 안) | 매 method 진입 / for 안 상태 | ✗ (개발만) |
| **DEBUG** | 디버깅용 상세 | request body / 외부 호출 detail | ✗ (개발 / staging) |
| **INFO** | 정상 흐름 이벤트 | request 시작 / state change / 외부 호출 결과 | ✓ |
| **WARN** | 즉시 실패는 아니지만 비정상 | 재시도 / 부하 / deprecated API 사용 | ✓ |
| **ERROR** | 즉시 실패 — 응답 실패 / 데이터 불일치 | 5xx / DB 에러 / 외부 API 실패 | ✓ |
| **FATAL** | 시스템 중단 | startup 실패 / 데이터 corruption | ✓ |

→ Logback 의 default level 은 `INFO`. production 에서 DEBUG 켜면 로그 폭주 + 성능 저하.

### 4.2 동적 변경

```yaml
# application.yml
logging:
  level:
    root: INFO
    com.example: DEBUG                # 본인 코드만 DEBUG
    org.springframework.web: INFO
    org.hibernate.SQL: DEBUG          # SQL 만 보고 싶을 때
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE   # SQL parameter
```

→ Spring Boot Actuator 의 `/actuator/loggers/{logger}` 로 runtime 변경 가능.

### 4.3 1 lecture 의 트리아지 예 ("CS 가 식당 12345 의 웨이팅 등록이 안 됐다고 함")

```bash
# 1. ERROR 만 먼저
grep "ERROR" app.log | grep "12345"

# 2. 발견된 request_id 로 INFO + WARN 까지
grep "request_id=a7c3" app.log | grep -E "INFO|WARN|ERROR"

# 3. 발견된 시점 5분 안의 모든 컨텍스트
awk '/10:20:00/,/10:25:00/' app.log | grep "request_id=a7c3"
```

→ MDC 의 request_id 가 있으면 grep 1 회로 끝. [[mdc-logging-filter]].

---

## 5. 1 줄 로그의 8 구성요소

```
2026-05-19 10:23:11.142 [http-nio-8080-exec-3] INFO  c.e.r.RestaurantController [a7c3...] - 식당 12345 웨이팅 등록 완료 - user=user_789 wait=15
       │                       │              │             │              │           │                                  │
       │                       │              │             │              │           │                                  └─ context (key=value)
       │                       │              │             │              │           └─ message
       │                       │              │             │              └─ MDC (request_id) — [[mdc-logging-filter]]
       │                       │              │             └─ logger name (보통 class)
       │                       │              └─ Level
       │                       └─ thread name
       └─ timestamp (ISO-8601 + ms)
```

| # | 요소 | 의미 | Logback pattern |
| --- | --- | --- | --- |
| 1 | timestamp | 언제 (ms 까지) | `%d{yyyy-MM-dd HH:mm:ss.SSS}` |
| 2 | thread | 어느 thread | `[%thread]` |
| 3 | level | 중요도 | `%-5level` |
| 4 | logger | 출처 (class) | `%logger{40}` |
| 5 | MDC | request id / user id | `[%X{request_id}]` |
| 6 | message | 무엇 | `%msg` |
| 7 | context | key=value 추가 | (message 안 또는 JSON) |
| 8 | exception | stack trace (ERROR 시) | `%ex` |

### 5.1 Logback default pattern

```
%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n%ex
```

### 5.2 권장 — MDC 포함

```
%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{40} [%X{request_id:-}] - %msg%n%ex
```

→ `%X{key:-기본값}` — MDC 에 없으면 빈 문자열.

상세: [[logback-configuration]] §3.

---

## 6. SLF4J + Logback 의 표준 사용법

### 6.1 logger 선언

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Service
public class RestaurantService {
    private static final Logger log = LoggerFactory.getLogger(RestaurantService.class);

    public void registerWaiting(Long restaurantId, Long userId) {
        log.info("식당 {} 웨이팅 등록 시작 - user={}", restaurantId, userId);
        try {
            // ...
            log.info("식당 {} 웨이팅 등록 완료 - user={} wait={}", restaurantId, userId, count);
        } catch (DuplicateWaitingException e) {
            log.warn("이미 등록된 웨이팅 - user={} restaurant={}", userId, restaurantId);
            throw e;
        } catch (Exception e) {
            log.error("웨이팅 등록 실패 - user={} restaurant={}", userId, restaurantId, e);
            throw e;
        }
    }
}
```

### 6.2 Lombok `@Slf4j`

```java
import lombok.extern.slf4j.Slf4j;

@Slf4j
@Service
public class RestaurantService {
    public void registerWaiting(...) {
        log.info("...", ...);          // log 변수 자동 생성
    }
}
```

### 6.3 parameterized message — `{}` 사용

```java
// GOOD — string 생성을 logger 가 level 검사 후 결정
log.debug("user {} 의 score 계산 중 - features={}", userId, features);

// BAD — string 항상 생성됨 (Level OFF 여도)
log.debug("user " + userId + " 의 score 계산 중 - features=" + features);
```

→ Level 이 OFF 면 `{}` 형식은 string concatenation 자체를 생략. 성능 이슈 차단.

### 6.4 예외 로깅

```java
// GOOD — exception 이 마지막 인자
log.error("결제 실패 - order={}", orderId, e);
// 출력: ... 결제 실패 - order=12345
//       java.lang.RuntimeException: ...
//           at ...

// BAD — exception 을 message 안 string 으로
log.error("결제 실패 - order=" + orderId + " ex=" + e.getMessage());
// stack trace 사라짐
```

### 6.5 Marker (선택)

특정 로그 type 을 별도 처리. 예: 비즈니스 audit log.

```java
import org.slf4j.Marker;
import org.slf4j.MarkerFactory;

private static final Marker AUDIT = MarkerFactory.getMarker("AUDIT");

log.info(AUDIT, "user {} 권한 변경 by admin {}", userId, adminId);
```

→ Logback 에서 marker 기반 filter / 다른 file appender 가능.

---

## 7. 로그 안티패턴

| 안티패턴 | 문제 | 정정 |
| --- | --- | --- |
| `e.printStackTrace()` 또는 `System.out.println` | logger 미사용 → Level / format / 수집 안 됨 | SLF4J logger |
| `log.error("...", e.getMessage())` | stack trace 사라짐 | exception 자체 마지막 인자 |
| string concatenation message | DEBUG OFF 여도 비용 | `{}` parameterized |
| catch + 로깅 + 다시 던지지도 무시도 안 함 | 흐름 불명 | rethrow 또는 명시적 처리 |
| 같은 예외 여러 layer 에서 로깅 | 중복 stack trace | 가장 바깥 layer 만 |
| 민감 데이터 로깅 (password / token / 카드) | 보안 사고 | 마스킹 / 제외 |
| 모든 method 진입에 INFO | 로그 폭주 | 의미 있는 이벤트만 / TRACE 로 |
| 큰 request body 전체 로깅 | 디스크 / 비용 | 필드 선택 / 길이만 |
| 동기 로깅으로 성능 저하 | I/O 가 thread 점유 | AsyncAppender (작은 trade-off) |
| 로그 회전 / 정리 안 함 | 디스크 가득 | RollingFileAppender + 정책 |
| 환경별 Level 같음 | dev/prod 의 noise 다름 | profile 별 application-{env}.yml |

---

## 8. 표준 로그 위치

| 위치 | 무엇 | Level |
| --- | --- | --- |
| Controller / Service 진입점 | request id / user id / 핵심 인자 | INFO |
| 외부 호출 직전 / 직후 | URL / 결과 / latency | INFO |
| state change (DB write) | 변경 대상 / 변경 후 값 | INFO |
| 검증 실패 (4xx) | 어떤 검증 / 입력 (마스킹) | WARN |
| 비즈니스 거부 (잔액 부족) | 어떤 정책 / 사용자 | WARN |
| 시스템 실패 (5xx / NPE) | exception + context | ERROR |
| startup 완료 / 종료 | 환경 변수 / version | INFO |
| Spring Bean 초기화 | 자동 (`@PostConstruct` 등) | DEBUG |

---

## 9. 측정 — 로그 품질 점검

| 항목 | 점검 방법 |
| --- | --- |
| 1 request 추적 가능? | request_id 로 grep — 모든 layer 의 로그가 같은 id 보유 |
| 평균 1 request 의 로그 수 | grep + count — 보통 5-15 줄 (너무 많으면 noise) |
| ERROR 비율 | grep ERROR / 전체 — 보통 0.1% 이하 |
| 디스크 사용량 | 일별 로그 크기 — 회전 정책 적정 검증 |
| 민감 데이터 노출 | grep `password|token|ssn|card` — 0 건 |

---

## 10. 흔한 실패 모드

| 실패 | 원인 | 정정 |
| --- | --- | --- |
| 로그가 전혀 안 보임 | Logback 미설정 (Spring Boot 는 기본 console 만) | application.yml + logback-spring.xml |
| 같은 메시지가 두 번 | logger 가 root + named 양쪽으로 흘러감 | `additivity=false` |
| DEBUG 켰는데 안 보임 | logger 의 effective level | `getLogger("...").isDebugEnabled()` 확인 |
| 비싼 logging 으로 성능 저하 | DEBUG 가 string concat | `{}` parameterized |
| stack trace 누락 | `e.getMessage()` 사용 | exception 자체 인자 |
| ERROR 가 100% 비정상 | ERROR 와 WARN 혼동 | Level 가이드 (§4.1) 검증 |
| production 에서 갑자기 로그 0 | Logback config error / SLF4J binding 실패 | `-Dlogback.debug=true` 로 startup 확인 |
| 로그 디스크 가득 | 회전 정책 없음 | RollingFileAppender — [[logback-configuration]] |

---

## 11. 참고

- [[logging|↑ logging]]
- [[mdc-logging-filter]] — request_id 부여
- [[logback-configuration]] — 파일 / 회전 / 압축
- [[elk-stack]] — 중앙 수집
- SLF4J 공식 — https://www.slf4j.org/
- Logback 공식 — https://logback.qos.ch/
- "Logging in Spring Boot" — https://docs.spring.io/spring-boot/reference/features/logging.html
