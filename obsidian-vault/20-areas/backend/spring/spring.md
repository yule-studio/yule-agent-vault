---
title: "Spring — Framework Hub"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T10:30:00+09:00
tags:
  - backend
  - spring
  - framework
  - hub
---

# Spring — Framework Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | hub — 언어별 (java/kotlin) 폴더 + 공통 개념 |

**[[../backend|↑ backend]]**

> Spring Framework / Boot / Security / Data JPA / Cloud / Batch / Cloud Gateway 의 **framework 공통 hub**.
> 언어 변형은 [[java-spring/java-spring|↗ java-spring]] · [[kotlin-spring/kotlin-spring|↗ kotlin-spring]].

---

## 1. 폴더 구조

```
spring/
├── spring.md                 ← 본 hub (framework 공통 / 비교)
├── java-spring/              ← Java + Spring Boot
│   ├── java-spring.md        ← 언어/스택 hub
│   ├── api-design/           ← API 구현 레시피 (signup / login / ...)
│   └── database/             ← Spring + Java 의 DB 통합 (Hikari / JPA / MyBatis)
└── kotlin-spring/            ← Kotlin + Spring Boot
    ├── kotlin-spring.md
    ├── api-design/           (추후)
    └── database/             (추후)
```

각 언어 폴더 내부 표준:
- `api-design/` — 회원가입·로그인·결제 등 **실전 API 레시피**. OOP 설계 + 코드 + 테스트.
- `database/` — PostgreSQL / MySQL / Redis 등 **Spring 진영에서의 통합** (Hikari 설정, JPA dialect, MyBatis mapper 등). DB 자체 운영은 [[../../database/database|↗ 20-areas/database]] 가 진실의 원천.

---

## 2. 한 줄 정의

**Spring** = Java/Kotlin 엔터프라이즈 백엔드의 사실상 표준.
2003 Spring 1.0 → 2014 Spring Boot 1.0 (Convention over Configuration) → 2022 Boot 3.0 (Jakarta EE 9+, JDK 17+) → 2024 Boot 3.3 (Native, Virtual Threads, Observability).

---

## 3. 핵심 모듈 (2026)

| 모듈 | 용도 |
| --- | --- |
| Spring Core / Context | DI / AOP / Bean lifecycle |
| Spring Boot | auto-configuration / starter / actuator |
| Spring MVC | 동기 HTTP API (`@RestController`) |
| Spring WebFlux | 비동기 / Reactive (Netty) |
| Spring Security | 인증 / 인가 / OAuth2 / SAML |
| Spring Data (JPA / MongoDB / Redis / Elasticsearch) | 영속 |
| Spring Batch | 대용량 배치 |
| Spring Cloud (Gateway / Config / OpenFeign) | MSA |
| Spring Integration / Messaging (Kafka / RabbitMQ) | 메시징 |

---

## 4. 언어 선택 — Java vs Kotlin

| 기준 | Java | Kotlin |
| --- | --- | --- |
| 학습 곡선 | 낮음 (생태계 도움말 풍부) | 중간 |
| 코드 양 | 길다 (record / pattern matching 으로 개선) | 짧다 (data class / null safety) |
| Spring 호환 | 100% (표준) | 100% (`kotlin-spring` plugin) |
| Null 안정성 | `@Nullable` annotation 기반 | 언어 차원 (`?` / `!!`) |
| 채용 풀 | 거대 | 작지만 성장 |
| 빌드 속도 | 빠름 | Kotlin compile 살짝 느림 |
| 권장 시점 | 대규모 팀 / 기존 생태계 | 새 프로젝트 / 작은 팀 / Android 공유 |

본 vault 는 양쪽 다 다루지만, 우선순위는 **Java**.

---

## 5. Spring 진영 표준 의존성 (Boot 3.3)

```kotlin
// build.gradle.kts
plugins {
    java                                              // 또는 kotlin("jvm")
    id("org.springframework.boot") version "3.3.0"
    id("io.spring.dependency-management") version "1.1.5"
}
java { toolchain.languageVersion = JavaLanguageVersion.of(21) }

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    runtimeOnly("org.postgresql:postgresql")
    implementation("org.flywaydb:flyway-core")
    implementation("org.flywaydb:flyway-database-postgresql")
}
```

언어별 코드 컨벤션과 추가 라이브러리는 각 언어 hub 참고.

---

## 6. 어떤 영역을 어디서 본다

| 알고 싶은 것 | 위치 |
| --- | --- |
| Java + Spring Boot 시작 / 설정 | [[java-spring/java-spring]] |
| Kotlin + Spring Boot 시작 / 설정 | [[kotlin-spring/kotlin-spring]] |
| **회원가입/로그인/결제 등 API 구현** | [[java-spring/api-design/api-design]] |
| Hikari / Datasource / JPA 설정 | `java-spring/database/` (작성 예정) |
| MyBatis 매퍼 | `java-spring/database/` (작성 예정) |
| Spring Security 깊이 | `java-spring/security/` (작성 예정) |
| **DB 자체** (튜닝 / 백업 / 보안) | [[../../database/database\|↗ 20-areas/database]] |

---

## 7. Spring 버전 / Java 버전 매트릭스 (2026)

| Spring Boot | Spring | Java 최소 | Jakarta EE | 상태 |
| --- | --- | --- | --- | --- |
| 2.7.x | 5.3.x | 8 / 17 | EE 8 (javax) | EOL (commercial 만) |
| 3.0.x ~ 3.2.x | 6.0 ~ 6.1 | **17** | EE 9+ (jakarta) | 보안 패치 |
| **3.3.x** | 6.1.x | **17** | EE 10 | **현재 권장** |
| 3.4.x | 6.2.x | 17 | EE 10 | 신규 (2024) |

> **함정**: 2.x → 3.x 마이그레이션은 `javax.*` → `jakarta.*` 패키지 변경 필수. 도구: OpenRewrite recipe `org.openrewrite.java.spring.boot3.SpringBoot3BestPractices`.

---

## 8. 학습 자료

- **Spring docs** — docs.spring.io (1순위)
- **Spring Guides** — spring.io/guides (작은 hands-on)
- **Spring in Action** (6th) — Walls
- **Pro Spring 6** — Cosmina
- **Spring Boot in Practice** — Sharma
- **Baeldung** — baeldung.com (실전 검색)
- **Spring Academy** — academy.spring.io (Tanzu, 영상)

---

## 9. 관련

- [[java-spring/java-spring|↗ java-spring]] · [[java-spring/api-design/api-design|↗ Java Spring API 레시피]]
- [[kotlin-spring/kotlin-spring|↗ kotlin-spring]]
- [[../../database/database|↗ database hub]] (DB 자체)
- [[../../security/security|↗ security hub]]
- [[../backend|↑ backend]]
