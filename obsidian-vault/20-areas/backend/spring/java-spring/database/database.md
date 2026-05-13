---
title: "Java Spring — Data Access / ORM Hub"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T15:30:00+09:00
tags:
  - backend
  - java-spring
  - database
  - orm
  - hub
---

# Java Spring — Data Access / ORM Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | hub — ORM 선택 / Hikari / 결정 트리 |

**[[../java-spring|↑ Java Spring]]**

> **레시피 (`api-design/`) 에는 Repository port (도메인 interface) 까지만**.
> 구체 ORM 구현은 이 폴더의 노트가 본거지. JPA / MyBatis / 둘 다 / JOOQ 등.
> DB 자체 운영 (PG/MySQL 튜닝·보안·백업) 은 [[../../../../database/database|↗ 20-areas/database]].

---

## 1. Java Spring 의 Data Access 4가지

| 도구 | 한 줄 | 강점 | 약점 |
| --- | --- | --- | --- |
| **Spring Data JPA / Hibernate** | 객체-관계 매핑 (ORM) | 도메인 모델링 / 영속성 컨텍스트 / 마법 많음 | N+1 / OSIV / 학습 곡선 |
| **MyBatis** | SQL 직접 + 결과 매핑 | SQL 그대로 / 복잡 쿼리 / 튜닝 쉬움 | DTO 양산 / 도메인 모델 어려움 |
| **JOOQ** | 타입 안전 SQL DSL | 컴파일 시 SQL 검증 / Java 21 record | 라이선스 (PG/MySQL 무료) / 학습 |
| **JdbcTemplate / NamedParameterJdbcTemplate** | 가장 얇은 wrapper | 단순 / 명확 | boilerplate 많음 |

---

## 2. 의사결정 트리

```
Q1. 도메인 모델 중심 / Aggregate-DDD ?
├─ Yes → JPA (Spring Data JPA)
└─ No (CRUD + 보고서 위주) →

Q2. SQL 직접 컨트롤 + 복잡 쿼리 빈도?
├─ 매우 높음 → MyBatis 또는 JOOQ
└─ 보통 →

Q3. 타입 안전 + 컴파일 검증 원함?
├─ Yes → JOOQ
└─ No  → MyBatis

Q4. 가벼운 (CLI / 배치 / 매우 단순) ?
├─ Yes → JdbcTemplate
```

### 실전 패턴 (한국 SaaS 표준)

| 시나리오 | 권장 |
| --- | --- |
| 신규 e-commerce / SaaS | **JPA** 기본 + **MyBatis** 보고서·복잡 쿼리 |
| 레거시 SI / 거대 SQL 자산 | **MyBatis** 전체 |
| OLAP / 분석 / 통계 | **JOOQ** 또는 **MyBatis** |
| 마이크로서비스 stateless 함수 | **JdbcTemplate** 또는 **JPA** |
| 도메인이 거의 없는 admin tool | **JdbcTemplate** |

→ 한국에선 **JPA + MyBatis 공존** 이 가장 흔한 production 패턴.

상세 가이드:
- [[jpa]] — JPA 본편
- [[mybatis]] — MyBatis 본편
- [[jpa-mybatis-coexist]] — 둘 다 사용 시 세팅 / 패턴

---

## 3. JPA vs MyBatis — 1:1 비교

| 항목 | JPA | MyBatis |
| --- | --- | --- |
| **SQL** | 자동 생성 (JPQL/HQL) | 직접 작성 |
| **객체 매핑** | Entity 클래스 + 어노테이션 | mapper XML / @Select |
| **변경 감지 (dirty checking)** | ✅ 자동 | ❌ 명시적 UPDATE |
| **영속성 컨텍스트 / 1차 캐시** | ✅ | ❌ |
| **연관관계 (@OneToMany ...)** | ✅ (단, N+1 함정) | ❌ (수동 join + 매핑) |
| **트랜잭션** | `@Transactional` | `@Transactional` |
| **마이그레이션 (Flyway)** | 같음 | 같음 |
| **복잡 동적 SQL** | Specification / QueryDSL / @Query | XML `<if>`, `<choose>`, `<foreach>` |
| **벤치 단순 read** | 약간 느림 (proxy / dirty check 비용) | 빠름 |
| **벤치 복잡 update** | ✅ batch / cascade | 빠름 (직접 제어) |
| **DTO Projection** | `new ...()` JPQL / interface | mapper resultMap |
| **학습 곡선** | 가파름 | 완만 |
| **튜닝** | 마법 많음 → 디버깅 어려움 | SQL 그대로 → 명확 |

→ **JPA 도메인 / MyBatis 보고서** 가 자주 보이는 분담.

---

## 4. DataSource / HikariCP

ORM 무관 — DB 연결 풀은 **HikariCP** (Spring Boot 기본).

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:5432/shop
    username: ${DB_USER}
    password: ${DB_PASSWORD}
    hikari:
      pool-name: ShopHikariCP
      maximum-pool-size: 30
      minimum-idle: 10
      connection-timeout: 3000       # ms - 풀에서 connection 못 받으면 fail
      idle-timeout: 600000            # 10m
      max-lifetime: 1800000           # 30m (DB max < 이 값)
      validation-timeout: 5000
      leak-detection-threshold: 30000   # 30s 이상 hold = 경고 (dev/staging)
      data-source-properties:
        cachePrepStmts: true
        prepStmtCacheSize: 250
        prepStmtCacheSqlLimit: 2048
```

### 4.1 pool size 산정

> Hikari 의 공식 — `pool_size = ((core_count * 2) + effective_spindle_count)` 가 시작점. 단 PG 의 `max_connections` 와 함께 봐야 함.

- 외부 API 호출 / 비동기 IO 가 많으면 동시 트랜잭션 수 ↑ → pool size ↑
- 모든 인스턴스의 pool 합계 ≤ PG `max_connections - reserved`
- PgBouncer 도입 시 — Hikari pool size ↓↓ + PgBouncer 가 다중 인스턴스 정리

### 4.2 함정

- **`maxLifetime` ≥ DB `wait_timeout`**: DB 가 끊은 connection 을 Hikari 가 알아채지 못함 → 첫 query 에서 `Communications link failure`
- **`connectionTimeout` 너무 김**: 외부 폭증 시 길게 hang → cascade 실패
- **`leakDetectionThreshold`**: 운영엔 끄거나 매우 크게 (false positive)

---

## 5. 마이그레이션 — Flyway (ORM 무관)

```kotlin
// build.gradle.kts
implementation("org.flywaydb:flyway-core")
implementation("org.flywaydb:flyway-database-postgresql")
```

```
src/main/resources/db/migration/
├── V1__create_users.sql
├── V2__create_products.sql
├── V3__create_orders.sql
└── V20240514_001__add_phone.sql        # 협업 시 충돌 줄이려 날짜 prefix
```

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate              # 운영 — 절대 update / create-drop X
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: false
    validate-on-migrate: true
```

> **함정**: `ddl-auto: update` 는 dev 도 비추. **Flyway 가 단일 진실의 원천**. JPA 는 검증만.

---

## 6. 이 폴더의 노트

```
database/
├── database.md                  ← 본 hub
├── jpa.md                       ← Spring Data JPA / Hibernate 본편
├── mybatis.md                   ← MyBatis 본편
├── jpa-mybatis-coexist.md       ← 둘 다 사용 시 세팅 / 패턴
├── jooq.md                      (예정)
├── jdbctemplate.md              (예정)
└── multi-datasource.md          (예정 — 여러 DB 동시 사용)
```

진행:

| 노트 | 상태 |
| --- | --- |
| [[jpa]] | ✅ |
| [[mybatis]] | ✅ |
| [[jpa-mybatis-coexist]] | ✅ |
| jooq | 🟡 |
| jdbctemplate | 🟡 |
| multi-datasource | 🟡 |

---

## 7. 레시피와의 연결

[[../api-design/api-design|↗ api-design]] 의 각 레시피는 **Repository port (interface)** 만 정의. 실제 ORM 구현은:

| 레시피 | port | 권장 구현 |
| --- | --- | --- |
| [[../api-design/signup]] | `UserRepository` | JPA — [[jpa#user-aggregate-적용 예]] |
| [[../api-design/login-jwt]] | `RefreshTokenRepository` | JPA 또는 Redis (둘 다 [[jpa]] / [[../api-design/login-jwt]]) |
| [[../api-design/product-crud]] | `ProductRepository` | JPA — 트랜잭션 / 관계 풍부 |
| [[../api-design/product-search]] | (검색) | **JPA Specification** + 복잡한 동적 쿼리는 **MyBatis** |
| [[../api-design/order-stock]] | `OrderRepository` | JPA (낙관 락, 비관 락) |
| [[../api-design/payment-pg]] | `PaymentRepository` | JPA |

→ **JPA 가 도메인-중심 코어 / MyBatis 가 보고서 / 통계 / 복잡 검색**.

---

## 8. 운영 / 디버깅 체크리스트

- [ ] `spring.jpa.open-in-view = false` (OSIV 끄기 — [[jpa]] 참고)
- [ ] Hikari pool size + DB max_connections 매칭
- [ ] `p6spy` 또는 `datasource-proxy` (dev/staging)
- [ ] Hibernate `generate_statistics = true` (dev/staging)
- [ ] Flyway 마이그레이션 + JPA `validate`
- [ ] read replica 라우팅 시 AbstractRoutingDataSource
- [ ] `LazyInitializationException` / N+1 통합 테스트 ([[../pitfalls/n-plus-one]])
- [ ] connection leak detection (`leak-detection-threshold` dev 에만)

---

## 9. 관련

- [[../api-design/api-design|↗ API 레시피 hub]]
- [[../pitfalls/n-plus-one]] — ORM 의 첫 사고
- [[../pitfalls/pitfalls|↗ pitfalls hub]]
- [[../../../../database/database|↗ 20-areas/database (DB 자체)]]
- [[../java-spring|↑ Java Spring]]
