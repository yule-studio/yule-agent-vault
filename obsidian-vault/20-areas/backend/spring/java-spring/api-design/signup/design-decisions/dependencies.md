---
title: "권장 의존성 — build.gradle.kts"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:00:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - design-decisions
  - dependencies
  - gradle
---

# 권장 의존성 — build.gradle.kts

**[[design-decisions|↑ design-decisions hub]]**

> "어떤 라이브러리를 쓰나" — 각 의존성마다 왜 이걸 골랐는지.

---

## 1. 본 vault stack

| 카테고리 | 라이브러리 | 버전 |
| --- | --- | --- |
| Spring Boot | spring-boot-starter-* | 3.3.x |
| Java | OpenJDK | 21 LTS |
| Password hash | password4j | 1.8.2 |
| JWT | jjwt | 0.12.6 |
| AWS SES | aws sdk v2 | 2.25.50 |
| WebClient (NCP SENS) | spring-boot-starter-webflux | 3.3.x |
| Apple JWT 검증 | nimbus-jose-jwt | 9.40 |
| TOTP | googleauth | 1.5.0 |
| QR code | zxing core | 3.5.3 |
| Rate limit | bucket4j-redis | 8.10.1 |
| Distributed lock | shedlock | 5.13.0 |
| Database driver | postgresql | 42.7.x |
| Migration | flyway-core | 10.x |

---

## 2. 전체 build.gradle.kts

```kotlin
plugins {
    id("org.springframework.boot") version "3.3.5"
    id("io.spring.dependency-management") version "1.1.6"
    kotlin("jvm") version "1.9.25" apply false        // Java only
    id("java")
}

group = "com.example.shop"
version = "0.1.0"
java.sourceCompatibility = JavaVersion.VERSION_21

repositories {
    mavenCentral()
}

dependencies {
    // ─── Spring Boot Core ─────────────────────────────────────────
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springframework.boot:spring-boot-starter-security")

    // ─── 데이터 / DB ─────────────────────────────────────────────
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
    implementation("org.mybatis.spring.boot:mybatis-spring-boot-starter:3.0.3")
    implementation("org.flywaydb:flyway-core:10.18.0")
    implementation("org.flywaydb:flyway-database-postgresql:10.18.0")
    runtimeOnly("org.postgresql:postgresql:42.7.3")

    // ─── 인증 / 보안 ──────────────────────────────────────────────
    implementation("com.password4j:password4j:1.8.2")                  // Argon2id
    implementation("io.jsonwebtoken:jjwt-api:0.12.6")                  // JWT
    runtimeOnly("io.jsonwebtoken:jjwt-impl:0.12.6")
    runtimeOnly("io.jsonwebtoken:jjwt-jackson:0.12.6")
    implementation("com.nimbusds:nimbus-jose-jwt:9.40")                // Apple JWKS
    implementation("com.warrenstrange:googleauth:1.5.0")               // TOTP
    implementation("com.google.zxing:core:3.5.3")                       // QR

    // ─── 외부 통신 ─────────────────────────────────────────────────
    implementation("software.amazon.awssdk:sesv2:2.25.50")              // SES
    implementation("org.springframework.boot:spring-boot-starter-webflux") // WebClient

    // ─── Rate Limit / Lock ─────────────────────────────────────────
    implementation("com.bucket4j:bucket4j-redis:8.10.1")
    implementation("net.javacrumbs.shedlock:shedlock-spring:5.13.0")
    implementation("net.javacrumbs.shedlock:shedlock-provider-jdbc-template:5.13.0")

    // ─── 유틸 ──────────────────────────────────────────────────────
    implementation("com.github.f4b6a3:ulid-creator:5.2.3")              // ULID
    compileOnly("org.projectlombok:lombok:1.18.34")
    annotationProcessor("org.projectlombok:lombok:1.18.34")

    // ─── API Docs ─────────────────────────────────────────────────
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0")

    // ─── Test ──────────────────────────────────────────────────────
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.security:spring-security-test")
    testImplementation("org.testcontainers:postgresql:1.20.2")
    testImplementation("org.testcontainers:junit-jupiter:1.20.2")
    testImplementation("io.rest-assured:rest-assured:5.5.0")
}
```

---

## 3. 각 의존성의 "왜" — 4구조

### 3.1 Spring Boot 3.3.x

**왜**
- 2024년 최신 stable (3.3.5 = 2024-09).
- Java 17/21 baseline (3.x 부터).
- Spring 6 + Hibernate 6 + Jakarta EE 9 (javax → jakarta).

**왜 안 됨 (3.2 이하)**
- Hibernate 6.4+ 의 개선 (JdbcTypeCode for JSONB) 없음.
- Spring 6.1 의 virtual thread 지원 약함.

**트레이드오프**
- 일부 라이브러리 호환 X (구식 javax.* 의존). 점진적 마이그레이션 필요.

---

### 3.2 Java 21 LTS

**왜**
- LTS — 5년 long-term support.
- virtual thread (Project Loom) — Spring Boot 3.2+ 통합.
- record / sealed class / pattern matching 등 모든 modern 기능.

**왜 안 됨 (Java 17 LTS)**
- virtual thread 미지원.
- 점진 deprecation.

**왜 안 됨 (Java 22+)**
- non-LTS — 6개월마다 EOL.
- 운영 부담.

---

### 3.3 password4j 1.8.2

**왜**
- Argon2id 의 명시적 API.
- PHC 형식 (`$argon2id$v=19$...`) 자동 처리.
- `needsRehash` 패턴 우수.

**대안**
- Spring Security `Argon2PasswordEncoder` — 가능. 단 password4j 가 더 명시적.
- bouncycastle — 더 low-level.

자세히: [[password-hash]].

---

### 3.4 jjwt 0.12.6

**왜**
- 한국 / 글로벌 Spring 표준.
- 0.12+ fluent API 안정.
- `alg: "none"` 기본 거부.

**대안**
- Auth0 `java-jwt` — 비슷.
- Nimbus — 더 강력 (Apple JWKS 같은 곳에 사용).

**왜 둘 다 (jjwt + nimbus)**
- jjwt — 자체 JWT 발급 / 검증.
- nimbus — Apple / Google JWKS 같은 외부 IdP 의 RS256 검증.

---

### 3.5 AWS SES v2 SDK

**왜**
- 공식 — IAM / VPC / CloudTrail 통합.
- v2 SDK 가 비동기 / non-blocking 지원.

**대안**
- spring-mail (SMTP) — SES 의 SMTP endpoint 도 가능. 단 SDK 가 throughput ↑.

자세히: [[email-provider]].

---

### 3.6 WebFlux (WebClient)

**왜**
- NCP SENS / Kakao / Apple 등 REST API 호출.
- RestTemplate 은 maintenance only — WebClient 권장 (Spring 6).
- async / reactive — 대량 발송 시 throughput ↑.

**대안**
- OkHttp / Apache HttpClient — 가능. WebClient 가 Spring 통합 ↑.

---

### 3.7 nimbus-jose-jwt 9.40

**왜**
- Apple Sign In 의 RS256 JWKS 검증.
- jjwt 보다 외부 IdP 통합 강력.

**언제 사용**
- Apple / Google OIDC id_token 검증만.
- 자체 JWT 는 jjwt.

---

### 3.8 googleauth 1.5.0 (TOTP)

**왜**
- TOTP (RFC 6238) 표준 구현.
- QR URL 생성 함수 포함.

**대안**
- aerogear-otp-java — 비슷.
- 자체 구현 — RFC 6238 직접 따르면 가능하지만 라이브러리가 안전.

---

### 3.9 bucket4j-redis 8.10.1

**왜**
- Rate limit 표준 (token bucket).
- Redis backend — 분산 환경 동기.
- @RateLimit annotation 지원.

**대안**
- Resilience4j — 더 범용 (circuit breaker 등). rate limit 만 필요 시 bucket4j 가 단순.
- Spring AOP 자체 구현 — 복잡.

---

### 3.10 shedlock 5.13.0

**왜**
- 다중 인스턴스의 @Scheduled 단일 실행 보장.
- DB / Redis backend.
- 본 vault: JDBC backend (별도 인프라 X).

**대안**
- Quartz — 더 강력. 작은 작업엔 과잉.

---

### 3.11 ulid-creator 5.2.3

**왜**
- ULID 표준 구현.
- monotonic API.

**대안**
- 자체 구현 — gettable.
- Crockford32 직접 — error-prone.

자세히: [[../database/id-strategy]].

---

### 3.12 lombok

**왜**
- @Getter / @Builder / @Slf4j 등 보일러플레이트 ↓.

**왜 record 와 같이 사용**
- record = VO / DTO.
- lombok = JPA entity / Service (mutable).

**왜 일부는 lombok X**
- 도메인 (User aggregate) — 직접 메서드 (도메인 로직 명시).
- lombok 의 @Data / @AllArgsConstructor 는 도메인 위반.

---

### 3.13 springdoc-openapi 2.6.0

**왜**
- Spring 6 / Jakarta 호환.
- 옛 springfox 는 사실상 abandoned.

자세히: [[../../api-docs/swagger]].

---

### 3.14 Testcontainers

**왜**
- 통합 테스트가 진짜 PostgreSQL / Redis 사용.
- 매 테스트마다 깨끗한 환경.

**대안**
- H2 — schema 호환 부분 (lower(email) 같은 PG-specific).
- 공용 DB — flaky / 의존성.

---

## 4. 버전 핀 정책

### 4.1 왜 명시적 버전

```kotlin
implementation("com.password4j:password4j:1.8.2")        // ✅ 명시
implementation("com.password4j:password4j")              // ❌ Spring BOM 의존
```

**왜**
- Spring BOM 이 모든 라이브러리 버전 관리 X.
- 명시적 버전 = 재현 가능 + 보안 audit.

### 4.2 Renovate / Dependabot

- 자동 PR — 신규 버전 알림.
- 매주 review + merge.

---

## 5. 의존성 보안

### 5.1 OWASP Dependency Check

```kotlin
plugins {
    id("org.owasp.dependencycheck") version "9.0.9"
}
```

```bash
./gradlew dependencyCheckAnalyze
```

→ CVE DB 와 대조 → 취약점 보고.

### 5.2 왜 필수

- Log4Shell (2021) 사고 — 모든 Spring Boot 영향.
- 의존성 transitive 까지 추적 (직접 의존 + 그것이 의존하는 것 다).

---

## 6. 함정 모음

### 함정 1 — Spring Boot 2.x 잔존
javax → jakarta 마이그레이션 미진 → Spring 6 호환 X.
→ 3.x 로 일괄 마이그레이션.

### 함정 2 — Java 8 / 11 사용
record / sealed / virtual thread 등 모던 기능 X.
→ Java 21 LTS.

### 함정 3 — bcrypt 만 사용 (Argon2id 없음)
GPU 공격 저항 약함.
→ password4j + Argon2id.

### 함정 4 — RestTemplate 사용
maintenance only — 신규 기능 X.
→ WebClient.

### 함정 5 — 자체 JWT 구현
보안 사고 위험 (`alg: "none"` 등).
→ jjwt 사용.

### 함정 6 — H2 in-memory 로 통합 테스트
PG-specific 기능 (CHECK 제약 / partial index) 미검증.
→ Testcontainers.

### 함정 7 — 버전 핀 안 함
새 버전 자동 적용 시 회귀.
→ 명시 + Renovate / Dependabot.

### 함정 8 — 의존성 보안 점검 없음
Log4Shell 같은 사고 무방비.
→ OWASP Dependency Check + 매주 review.

### 함정 9 — Lombok 남용 (도메인에도)
@Data / @AllArgsConstructor 가 도메인 invariant 깨짐.
→ VO / Service 만, 도메인 aggregate 는 X.

### 함정 10 — 옛 springfox / swagger-ui-2.x
maintenance only.
→ springdoc-openapi.

---

## 7. 다른 컨텍스트

### 7.1 MSA

```kotlin
// 추가
implementation("org.springframework.cloud:spring-cloud-starter-config")
implementation("org.springframework.cloud:spring-cloud-starter-netflix-eureka-client")
implementation("io.github.resilience4j:resilience4j-spring-boot3:2.2.0")
```

### 7.2 Kotlin

```kotlin
plugins {
    kotlin("jvm") version "1.9.25"
    kotlin("plugin.spring") version "1.9.25"
    kotlin("plugin.jpa") version "1.9.25"
}
```

### 7.3 글로벌 (i18n)

```kotlin
implementation("org.springframework.boot:spring-boot-starter-thymeleaf")
implementation("com.ibm.icu:icu4j:75.1")              // i18n 강력 지원
```

---

## 8. 관련

- [[design-decisions|↑ hub]]
- [[default-stack]] — yaml config
- [[password-hash]] · [[token-model]] · [[email-provider]] · [[sms-provider]] — 각 의존성의 사용
- [[../../database/jpa]] · [[../../database/mybatis]]
- 외부 — Spring Boot Reference, OWASP Dependency Check
