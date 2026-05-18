---
title: "logging/logback-configuration — Appender / 파일 / 압축 / RollingFile"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-19T06:45:00+09:00
tags: [backend, java-spring, observability, logging, logback]
home_hub: logging
related:
  - "[[logging]]"
  - "[[fundamentals]]"
  - "[[mdc-logging-filter]]"
  - "[[elk-stack]]"
---

# logging/logback-configuration — Appender / 파일 / 압축 / RollingFile

**[[logging|↑ logging]]** · **[[../observability|↑ observability]]**

> 강의 매핑: §4 (로그를 파일로 / Appender / Logback 설정 / 압축).

---

## 1. 목적

본 문서는 Spring Boot 의 Logback 설정 — Appender / Encoder / Rolling Policy / 환경별 분리 / JSON 출력 — 의 표준 설정을 정의한다.

본 문서가 정의하는 것:
- Logback 의 5 구성요소 (Logger / Appender / Encoder / Layout / Filter)
- 4 Appender 종류 (Console / File / RollingFile / Async)
- RollingFileAppender 의 policy (time / size / time-and-size)
- 파일 압축 (`.gz`)
- 환경별 (dev / staging / prod) 분리
- JSON 출력 (logstash-logback-encoder) — ELK 사전 준비

본 문서가 정의하지 않는 것:
- 로그의 일반 사용 — [[fundamentals]]
- MDC / request_id — [[mdc-logging-filter]]
- ELK 자체 — [[elk-stack]]

---

## 2. Logback 의 5 구성요소

```
[Logger] ─── 어디 (어느 class) 의 로그
   │
   ├─ Level (TRACE / DEBUG / INFO / WARN / ERROR / FATAL)
   │
   └─→ [Appender] ─── 어디로 출력 (Console / File / 외부 시스템)
                  │
                  ├─ [Encoder] ─── 어떤 형식 (Pattern / JSON)
                  │
                  ├─ [Layout] ─── (구) 형식 정의 — Encoder 권장
                  │
                  └─ [Filter] ─── 어떤 로그만 통과 (Level / ThresholdFilter / EvaluatorFilter)
```

| 컴포넌트 | 책임 |
| --- | --- |
| Logger | logger 이름 (보통 class FQN). Level / Appender 할당. |
| Appender | 출력 destination — console / file / TCP / async wrapper |
| Encoder | bytes 로 변환 (PatternLayoutEncoder / LogstashEncoder) |
| Filter | 통과 / 거부 결정 |

---

## 3. Spring Boot 의 Logback 자동 설정

Spring Boot 는 `logback-spring.xml` (또는 `logback.xml`) 을 classpath 에서 자동 로드. 권장은 **`logback-spring.xml`** — Spring profile 사용 가능 (`<springProfile>` 태그).

### 3.1 디렉토리

```
src/main/resources/
├── application.yml
├── application-dev.yml
├── application-prod.yml
└── logback-spring.xml          ← 본 영역의 메인
```

### 3.2 application.yml 의 logging 설정

가벼운 설정은 yml 만으로:

```yaml
logging:
  level:
    root: INFO
    com.example: DEBUG
    org.springframework: INFO
    org.hibernate.SQL: DEBUG
  pattern:
    console: "%d{HH:mm:ss.SSS} [%thread] %-5level [%X{request_id:-}] %logger{40} - %msg%n"
    file: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level [%X{request_id:-}] %logger{40} - %msg%n"
  file:
    name: logs/application.log

  logback:
    rollingpolicy:
      file-name-pattern: logs/application-%d{yyyy-MM-dd}.%i.log.gz
      max-file-size: 100MB
      max-history: 30
      total-size-cap: 10GB
```

→ 위 yml 만으로 console + file + rolling + 압축 (.gz) 완성. 더 복잡한 (다중 appender / JSON / 환경별) 은 `logback-spring.xml` 필요.

---

## 4. logback-spring.xml — 표준 설정

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true" scanPeriod="30 seconds">

  <!-- Spring Boot 의 기본 패턴 / 색 등 inherit -->
  <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

  <!-- ============ 변수 ============ -->
  <property name="LOG_DIR" value="${LOG_DIR:-logs}"/>
  <property name="APP_NAME" value="${spring.application.name:-app}"/>
  <property name="PATTERN" value="%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level [%X{request_id:-}] %logger{40} - %msg%n%ex"/>

  <!-- ============ Console Appender ============ -->
  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>${PATTERN}</pattern>
    </encoder>
  </appender>

  <!-- ============ Rolling File Appender — application log ============ -->
  <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>${LOG_DIR}/${APP_NAME}.log</file>

    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
      <fileNamePattern>${LOG_DIR}/${APP_NAME}-%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
      <maxFileSize>100MB</maxFileSize>           <!-- 1 파일 최대 -->
      <maxHistory>30</maxHistory>                 <!-- 30 일치 보관 -->
      <totalSizeCap>10GB</totalSizeCap>           <!-- 총 한도 -->
    </rollingPolicy>

    <encoder>
      <pattern>${PATTERN}</pattern>
    </encoder>
  </appender>

  <!-- ============ Rolling File Appender — access log (Filter 별도) ============ -->
  <appender name="ACCESS_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>${LOG_DIR}/${APP_NAME}-access.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
      <fileNamePattern>${LOG_DIR}/${APP_NAME}-access-%d{yyyy-MM-dd}.log.gz</fileNamePattern>
      <maxHistory>30</maxHistory>
    </rollingPolicy>
    <encoder>
      <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%X{request_id:-}] %msg%n</pattern>
    </encoder>
  </appender>

  <!-- ============ Async wrapper — 성능 ============ -->
  <appender name="ASYNC_FILE" class="ch.qos.logback.classic.AsyncAppender">
    <appender-ref ref="FILE"/>
    <queueSize>1024</queueSize>
    <discardingThreshold>0</discardingThreshold>     <!-- 0 = 모든 로그 보존 -->
    <neverBlock>false</neverBlock>                   <!-- queue 가득 시 block (true=drop) -->
  </appender>

  <!-- ============ Logger 정의 ============ -->

  <!-- Access logger — application 과 분리 -->
  <logger name="ACCESS" level="INFO" additivity="false">
    <appender-ref ref="ACCESS_FILE"/>
    <appender-ref ref="CONSOLE"/>
  </logger>

  <!-- 본인 코드 -->
  <logger name="com.example" level="DEBUG"/>

  <!-- Spring noise 줄이기 -->
  <logger name="org.springframework.web" level="INFO"/>
  <logger name="org.hibernate.SQL" level="DEBUG"/>

  <!-- ============ root ============ -->
  <root level="INFO">
    <appender-ref ref="CONSOLE"/>
    <appender-ref ref="ASYNC_FILE"/>
  </root>

  <!-- ============ profile 별 추가 / 교체 ============ -->
  <springProfile name="prod">
    <root level="INFO">
      <!-- console 제외 (container 환경에서 stdout 도 file 로 처리됨) -->
      <appender-ref ref="ASYNC_FILE"/>
    </root>
  </springProfile>

  <springProfile name="dev">
    <logger name="com.example" level="TRACE"/>
  </springProfile>

</configuration>
```

| 핵심 옵션 | 의미 |
| --- | --- |
| `scan="true" scanPeriod="30 seconds"` | xml 변경 시 자동 reload — production 에서도 안전 |
| `<springProfile name="prod">` | profile 별 다른 설정 (Spring Boot 만) |
| `additivity="false"` | 부모 logger 로 전파 차단 — 중복 출력 방지 |
| `SizeAndTimeBasedRollingPolicy` | 시간 + 크기 동시 조건 |
| `.log.gz` extension | rollover 시 자동 gzip 압축 (Logback 내장) |
| AsyncAppender | I/O 가 thread 점유 안 함 — 성능 ↑ |

---

## 5. Appender 4 종류

### 5.1 ConsoleAppender

```xml
<appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
  <target>System.out</target>                     <!-- 또는 System.err -->
  <encoder>
    <pattern>${PATTERN}</pattern>
  </encoder>
</appender>
```

→ container / k8s 의 stdout 기반 로그 수집 표준.

### 5.2 FileAppender (단일 파일 — 비권장)

```xml
<appender name="FILE_SIMPLE" class="ch.qos.logback.core.FileAppender">
  <file>${LOG_DIR}/app.log</file>
  <encoder><pattern>${PATTERN}</pattern></encoder>
</appender>
```

→ 회전 없음. 무제한 증가. 실무 사용 X.

### 5.3 RollingFileAppender (표준)

#### TimeBasedRollingPolicy (시간만)

```xml
<appender name="FILE_TIME" class="ch.qos.logback.core.rolling.RollingFileAppender">
  <file>${LOG_DIR}/app.log</file>
  <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
    <fileNamePattern>${LOG_DIR}/app-%d{yyyy-MM-dd}.log.gz</fileNamePattern>
    <maxHistory>30</maxHistory>
    <totalSizeCap>5GB</totalSizeCap>
  </rollingPolicy>
  <encoder><pattern>${PATTERN}</pattern></encoder>
</appender>
```

→ 매일 / 매시 회전. 가장 단순.

#### SizeAndTimeBasedRollingPolicy (시간 + 크기 — 권장)

```xml
<rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
  <fileNamePattern>${LOG_DIR}/app-%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
  <maxFileSize>100MB</maxFileSize>
  <maxHistory>30</maxHistory>
  <totalSizeCap>10GB</totalSizeCap>
</rollingPolicy>
```

→ 매일 + 100MB 초과 시 동일 일의 다음 `.0`, `.1`, `.2` ... 생성. `%i` 가 index.

### 5.4 AsyncAppender (성능 wrapper)

```xml
<appender name="ASYNC_FILE" class="ch.qos.logback.classic.AsyncAppender">
  <appender-ref ref="FILE"/>
  <queueSize>1024</queueSize>
  <discardingThreshold>0</discardingThreshold>
  <neverBlock>false</neverBlock>
</appender>
```

| 옵션 | 의미 |
| --- | --- |
| queueSize | thread → queue 의 버퍼 크기 |
| discardingThreshold | queue 가 80% 가까우면 TRACE/DEBUG/INFO drop (기본 20%) |
| discardingThreshold=0 | 모든 로그 보존 (queue 가득이면 block / drop) |
| neverBlock=true | queue 가득 시 drop (log loss 허용) |
| neverBlock=false | queue 가득 시 block (request latency 증가) |

→ AsyncAppender 는 I/O 대기를 thread 에서 분리 — 성능 향상 but log loss 위험 trade-off.

### 5.5 기타 (참고)

| Appender | 사용 |
| --- | --- |
| SMTPAppender | 이메일 발송 (ERROR 알람) — 비권장 (현대는 Alert 시스템 사용) |
| DBAppender | DB 저장 — 비권장 (성능 이슈) |
| SocketAppender | TCP 로 외부 전송 (Logstash 의 대안) |
| SyslogAppender | syslog protocol |

---

## 6. 파일 회전 / 압축 / 보관

### 6.1 회전 시점

| 정책 | 회전 시점 |
| --- | --- |
| TimeBasedRollingPolicy | `%d{yyyy-MM-dd}` 의 단위 변경 (매일) / `%d{yyyy-MM-dd-HH}` (매시) |
| SizeAndTimeBased | 시간 단위 변경 + maxFileSize 초과 (둘 중 빠른 쪽) |

### 6.2 압축 — 자동

`fileNamePattern` 이 `.gz` 또는 `.zip` 으로 끝나면 Logback 이 회전 시 자동 압축.

```xml
<fileNamePattern>${LOG_DIR}/app-%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
```

→ 100MB 파일 → ~5-10MB 로 감소. 디스크 90% 절약.

### 6.3 보관 정책

| 옵션 | 의미 |
| --- | --- |
| `maxHistory` | 보관 일수 (또는 회전 단위 수) |
| `totalSizeCap` | 디렉토리 총 크기 한도 — 초과 시 가장 오래된 파일 자동 삭제 |
| `cleanHistoryOnStart` | 시작 시 정책 위반 파일 정리 |

→ 디스크 가득 사고 방지의 핵심. **모든 production logger 는 `totalSizeCap` 명시**.

---

## 7. JSON 출력 — logstash-logback-encoder (ELK 사전 준비)

### 7.1 의존성

```kotlin
// build.gradle.kts
implementation("net.logstash.logback:logstash-logback-encoder:8.0")
```

### 7.2 JSON FileAppender — 파일에 JSON 으로

```xml
<appender name="JSON_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
  <file>${LOG_DIR}/${APP_NAME}-json.log</file>

  <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
    <fileNamePattern>${LOG_DIR}/${APP_NAME}-json-%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
    <maxFileSize>100MB</maxFileSize>
    <maxHistory>30</maxHistory>
    <totalSizeCap>10GB</totalSizeCap>
  </rollingPolicy>

  <encoder class="net.logstash.logback.encoder.LogstashEncoder">
    <includeMdcKeyName>request_id</includeMdcKeyName>
    <includeMdcKeyName>user_id</includeMdcKeyName>
    <includeMdcKeyName>tenant_id</includeMdcKeyName>
    <customFields>{"service":"${APP_NAME}","env":"${ENV:-dev}"}</customFields>
  </encoder>
</appender>
```

### 7.3 출력 예

```json
{
  "@timestamp": "2026-05-19T10:23:11.142+09:00",
  "@version": "1",
  "message": "식당 12345 웨이팅 등록 완료",
  "logger_name": "com.example.RestaurantService",
  "thread_name": "http-nio-8080-exec-3",
  "level": "INFO",
  "level_value": 20000,
  "request_id": "a7c3b421",
  "user_id": "user_789",
  "service": "restaurant-api",
  "env": "prod"
}
```

→ 이 파일을 Filebeat 또는 Logstash 가 읽어 Elasticsearch 로. [[elk-stack]] §5.

### 7.4 Logstash TCP 직접 전송 (사전 준비)

```xml
<appender name="LOGSTASH_TCP" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
  <destination>logstash:5000</destination>
  <encoder class="net.logstash.logback.encoder.LogstashEncoder">
    <includeMdcKeyName>request_id</includeMdcKeyName>
    <customFields>{"service":"${APP_NAME}"}</customFields>
  </encoder>
  <reconnectionDelay>1 second</reconnectionDelay>
  <writeBufferSize>16384</writeBufferSize>
</appender>
```

→ TCP 단절 시 자동 재연결. application 측 buffer 로 단기 단절 흡수.

---

## 8. 환경별 분리 — `<springProfile>`

```xml
<configuration>
  <!-- 공통 -->
  ...

  <springProfile name="dev">
    <logger name="com.example" level="TRACE"/>
    <root level="DEBUG">
      <appender-ref ref="CONSOLE"/>
    </root>
  </springProfile>

  <springProfile name="staging">
    <logger name="com.example" level="DEBUG"/>
    <root level="INFO">
      <appender-ref ref="CONSOLE"/>
      <appender-ref ref="ASYNC_FILE"/>
    </root>
  </springProfile>

  <springProfile name="prod">
    <root level="INFO">
      <appender-ref ref="ASYNC_FILE"/>
      <appender-ref ref="JSON_FILE"/>             <!-- ELK 용 -->
    </root>
  </springProfile>
</configuration>
```

→ `--spring.profiles.active=prod` 로 활성화. 컨테이너 환경에서 `SPRING_PROFILES_ACTIVE=prod`.

---

## 9. 외부에서 변경 — Spring Boot Actuator

`spring-boot-starter-actuator` 의 `/actuator/loggers` endpoint 로 runtime 에 Level 변경.

```bash
# 현재 level 조회
curl http://localhost:8080/actuator/loggers/com.example

# 동적 변경 — 재시작 없이 DEBUG 켜기
curl -X POST http://localhost:8080/actuator/loggers/com.example \
  -H "Content-Type: application/json" \
  -d '{"configuredLevel":"DEBUG"}'
```

→ production 의 디버깅 시 매우 강력. 보안 — Actuator endpoint 는 internal 또는 인증 필수.

---

## 10. 디스크 / 성능 — 측정

| 항목 | 측정 |
| --- | --- |
| 일별 로그 크기 | `du -sh logs/` 또는 일자별 ls -lh |
| 압축률 | `.log.gz` vs `.log` 크기 비교 — 보통 5-15% |
| AsyncAppender queue 가득 빈도 | `LogbackMetrics` (Micrometer) — `logback.events.dropped` |
| logger 호출 평균 latency | wrapper benchmark (보통 < 10μs) |

---

## 11. 흔한 실패 모드

| 실패 | 원인 | 정정 |
| --- | --- | --- |
| 로그 파일이 무한 증가 | rolling 없음 | RollingFileAppender + totalSizeCap |
| 같은 로그가 두 번 | additivity 누락 | `additivity="false"` |
| AsyncAppender 가 INFO drop | discardingThreshold 기본 20% | `discardingThreshold=0` |
| 회전 시 application 정지 | sync I/O | AsyncAppender |
| 파일이 새 일자에 안 회전 | fileNamePattern 의 `%d` 누락 | pattern 검증 |
| Tomcat thread 종료 후에도 MDC 잔존 (다른 logger) | clear 안 함 | [[mdc-logging-filter]] §10 |
| JSON 출력에 MDC 안 보임 | `includeMdcKeyName` 미설정 | provider 명시 |
| profile 별 설정 안 적용 | `<springProfile>` 의 자식 만 적용됨 | profile 안 root / logger 별도 정의 |
| `logback.xml` (Spring profile 무시) 사용 | `logback-spring.xml` 권장 | 파일명 변경 |
| `logback.debug=true` 로 startup 디버그 | scan 작동 안 함 | scanPeriod 검증 |
| 디스크 / 메모리 사용 폭주 | DEBUG / TRACE production 켰음 | logger Level 검증 |

---

## 12. 적용 체크 항목

| 항목 | 점검 |
| --- | --- |
| `logback-spring.xml` 사용 (logback.xml 아님) |  |
| RollingFileAppender + totalSizeCap |  |
| `.log.gz` extension 으로 자동 압축 |  |
| `<springProfile>` 으로 환경별 분리 |  |
| `additivity="false"` 명시 (별도 logger) |  |
| AsyncAppender wrap (성능 critical) |  |
| MDC pattern 포함 — `%X{request_id:-}` |  |
| (ELK 사용 시) LogstashEncoder + JSON file 또는 TCP |  |
| Actuator `/loggers` 보안 (인증 / 내부 only) |  |
| `scan="true"` 로 핫리로드 가능 |  |

---

## 13. 참고

- [[logging|↑ logging]]
- [[fundamentals]]
- [[mdc-logging-filter]]
- [[elk-stack]]
- Logback 공식 manual — https://logback.qos.ch/manual/
- Spring Boot Logging — https://docs.spring.io/spring-boot/reference/features/logging.html
- logstash-logback-encoder — https://github.com/logfellow/logstash-logback-encoder
- Effective Logback (블로그 시리즈)
