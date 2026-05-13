---
title: "JPA + MyBatis 공존 — 같이 쓰는 표준 패턴"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T16:15:00+09:00
tags:
  - backend
  - java-spring
  - database
  - jpa
  - mybatis
  - coexistence
---

# JPA + MyBatis 공존 — 같이 쓰는 표준 패턴

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 단일 DataSource / TransactionManager 통합 / 역할 분담 / 동기화 함정 |

**[[database|↑ database hub]]**

> 전제 — [[jpa]] / [[mybatis]] 본편 먼저.

---

## 1. 왜 같이 쓰나

| 영역 | 더 잘 맞는 도구 | 이유 |
| --- | --- | --- |
| **도메인 CUD (Create/Update/Delete)** | **JPA** | 영속성 컨텍스트 / dirty checking / Aggregate 일관성 / 트랜잭션 안전 |
| **단순 조회 (PK / 작은 필터)** | **JPA** | Repository 표준 메서드 |
| **복잡 동적 SQL (검색 / 보고서 / 통계)** | **MyBatis** | SQL 가독성 / 튜닝 직접 |
| **다중 테이블 JOIN + 통계** | **MyBatis** | aggregate / projection 자유 |
| **레거시 SQL 자산 이관** | **MyBatis** | 그대로 |

→ 한국 SaaS 표준 패턴.

### 1.1 언제 안 같이 쓰는 게 좋나

- 신규 프로젝트 / 도메인 단순 → **JPA 만**
- 거대 레거시 / SQL 자산 / 도메인 모델 약함 → **MyBatis 만**
- **MSA 마이크로서비스**: 작아서 한 가지로 충분
- 팀이 어느 한쪽만 익숙하면 학습 비용 ↑

---

## 2. 핵심 원칙

1. **단일 DataSource** — 둘이 같은 connection pool 공유
2. **단일 TransactionManager** — JPA `JpaTransactionManager` 가 MyBatis 까지 처리
3. **JPA 가 dirty checking 의 진실의 원천** — 같은 트랜잭션에서 JPA 로 변경 → MyBatis `SELECT` = **flush 필요**
4. **역할 분리** — 같은 테이블의 CUD 를 양쪽에서 쓰지 말 것

---

## 3. 설정 — 단일 DataSource + 단일 TransactionManager

```kotlin
// build.gradle.kts
implementation("org.springframework.boot:spring-boot-starter-data-jpa")
implementation("org.mybatis.spring.boot:mybatis-spring-boot-starter:3.0.3")
runtimeOnly("org.postgresql:postgresql")
```

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/shop
    username: ${DB_USER}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 30
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        default_batch_fetch_size: 100

mybatis:
  mapper-locations: classpath:mapper/**/*.xml
  type-aliases-package: com.example.shop.domain
  configuration:
    map-underscore-to-camel-case: true
    default-statement-timeout: 5
```

> **그게 다** — Spring Boot 의 auto-configuration 이 같은 DataSource 를 두 starter 가 공유. 별도 설정 거의 없음.

### 3.1 트랜잭션 매니저

Spring Boot 가 `JpaTransactionManager` 자동 등록 → MyBatis 의 SqlSession 도 같은 트랜잭션에 들어감.

```java
@Service
public class SignupService {
    private final UserJpaRepository userJpa;       // JPA
    private final AuditMapper auditMapper;          // MyBatis

    @Transactional                                  // 둘 다 같은 트랜잭션
    public void signup(UserJpaEntity user, AuditLogRow audit) {
        userJpa.save(user);
        auditMapper.insert(audit);
        // 한쪽 실패 시 둘 다 rollback
    }
}
```

> Spring Boot 3 + MyBatis-Spring-Boot-Starter 3.0+ 는 자동. 별도 `SqlSessionFactory` 빈 정의 필요 없음.

---

## 4. 역할 분담 패턴 — 추천

### 4.1 동일 entity / 다른 read path

가장 자주 보는 패턴 — **JPA 가 CUD + 단순 read, MyBatis 가 검색 / 보고서**.

```java
// 도메인 port
public interface UserRepository {                  // CUD + 단순 조회
    User save(User user);
    Optional<User> findByEmail(Email email);
}
public interface UserQueryRepository {             // 복잡 검색 / 보고서
    Page<UserSearchResult> search(UserSearchCriteria criteria, Pageable pageable);
    List<UserMonthlyStats> monthlyStats(int year);
}
```

```java
// JPA — 쓰기 + 트랜잭션
@Repository
@RequiredArgsConstructor
public class JpaUserRepository implements UserRepository {
    private final UserJpaRepository spring;
    // save / findByEmail 구현
}

// MyBatis — 복잡 read
@Repository
@RequiredArgsConstructor
public class MyBatisUserQueryRepository implements UserQueryRepository {
    private final UserQueryMapper mapper;

    @Override
    public Page<UserSearchResult> search(UserSearchCriteria c, Pageable p) {
        var rows = mapper.search(c, p.getOffset(), p.getPageSize());
        var total = mapper.count(c);
        return new PageImpl<>(rows, p, total);
    }
}
```

→ 도메인은 두 interface 모두 모름 / 모든 application service 는 두 repository 를 받음.

### 4.2 같은 트랜잭션 안의 dirty read 함정

```java
@Transactional
public void doSomething(UserId id) {
    var user = userJpaRepo.findById(id.value());   // JPA 영속
    user.setName("changed");                        // 객체만 변경 (UPDATE 아직 아님)

    // 같은 트랜잭션에서 MyBatis 로 조회
    var row = userMapper.findById(id.value());      // ⚠️ DB 의 옛 name 이 보임
}
```

**원인**: JPA 의 dirty checking 은 **트랜잭션 커밋 시점** 에 UPDATE. MyBatis 는 그 변경을 모름.

**해결**: 같은 트랜잭션 안에서 JPA 로 변경 후 MyBatis 로 읽기 직전에 **`em.flush()`**:

```java
@Transactional
public void doSomething(UserId id) {
    var user = userJpaRepo.findById(id.value()).orElseThrow();
    user.setName("changed");
    em.flush();                                     // UPDATE 즉시 발사
    var row = userMapper.findById(id.value());      // 새 name 보임
}
```

**더 나은 규칙**: 같은 트랜잭션 안에서 **같은 테이블의 JPA 쓰기 + MyBatis 읽기를 섞지 말 것**. 가능하면 분리.

---

## 5. 같은 테이블, 다른 도구 — 안티패턴?

```
users 테이블:
  ├─ JPA: UserJpaEntity (CUD)
  └─ MyBatis: UserSearchMapper (read)
```

OK — read 와 write 가 분리됐고, MyBatis 는 select 만.

```
products 테이블:
  ├─ JPA: ProductJpaEntity (CUD)
  ├─ MyBatis: ProductReportMapper (집계 read)
  └─ MyBatis: ProductBulkUpdateMapper (✗ CUD)
```

→ **MyBatis 의 CUD 가 들어가면 JPA 의 영속성 컨텍스트와 충돌 위험**. 같은 트랜잭션에서:

```java
@Transactional
public void example() {
    var p = jpaRepo.findById(...).orElseThrow();   // 영속성 컨텍스트 진입
    mybatisMapper.bulkUpdateStatus(...);            // MyBatis 가 직접 UPDATE
    // JPA 영속성 컨텍스트의 p 는 옛 상태 그대로
    p.changeSomething();                             // 커밋 시 옛 상태로 UPDATE → MyBatis 변경 덮어쓰기
}
```

→ **JPA 가 캐싱한 entity 를 MyBatis 가 우회로 변경하면 사고**.

**규칙**:
1. 한 테이블의 **CUD 는 JPA 단일 담당**
2. MyBatis 가 같은 테이블 변경하려면 **새 트랜잭션** 또는 **JPA 컨텍스트 clear**

```java
@Transactional
public void bulkArchive() {
    mybatisMapper.bulkUpdateToArchive();           // 대량 UPDATE
    em.clear();                                     // 1차 캐시 비움
}
```

---

## 6. 페이지네이션 호환

JPA 의 `Pageable` 을 MyBatis 에서 활용:

```java
// MyBatis mapper
@Mapper
public interface UserMapper {
    @Select("SELECT ... FROM users WHERE ... LIMIT #{limit} OFFSET #{offset}")
    List<UserRow> search(@Param("criteria") UserSearchCriteria c,
                         @Param("limit") int limit,
                         @Param("offset") long offset);

    @Select("SELECT count(*) FROM users WHERE ...")
    long count(@Param("criteria") UserSearchCriteria c);
}

// Service
public Page<UserDto> search(UserSearchCriteria c, Pageable pageable) {
    var rows = mapper.search(c, pageable.getPageSize(), pageable.getOffset());
    var total = mapper.count(c);
    return new PageImpl<>(rows.stream().map(this::toDto).toList(),
                          pageable, total);
}
```

→ Controller 가 `Pageable` 받음. Service / Repository 가 JPA `Pageable` 을 MyBatis 인자로 변환.

---

## 7. 매핑 라이브러리 — MapStruct (선택)

JPA Entity / MyBatis Row / Domain / DTO 변환이 늘어남:

```kotlin
implementation("org.mapstruct:mapstruct:1.5.5.Final")
annotationProcessor("org.mapstruct:mapstruct-processor:1.5.5.Final")
```

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
    UserDto toDto(UserJpaEntity entity);
    UserDto toDto(UserRow row);
    UserJpaEntity toJpa(User domain);
    UserRow toRow(User domain);
}
```

→ boilerplate 매핑 줄임. 도메인 ↔ JPA / Row / DTO 4 종 매핑이 자주 필요한 큰 프로젝트에 권장.

---

## 8. 트랜잭션 / 격리 / 외부 호출

[[../api-design/payment-pg#5.1]] 처럼 — **PG 호출 / SMTP / Kafka 등 외부 IO 는 트랜잭션 밖**.

```java
public Result payment(...) {
    // (tx) — JPA + MyBatis 같은 트랜잭션에서 row insert
    tx.executeWithoutResult(s -> {
        paymentJpaRepo.save(payment);
        auditMapper.insert(audit);
    });

    // (no tx) — PG 외부 API
    var result = pg.confirm(...);

    // (tx) — 결과 반영
    tx.executeWithoutResult(s -> {
        payment.approve(...);
        paymentJpaRepo.save(payment);
    });
}
```

---

## 9. 디렉토리 / 패키지 구조

```
com.example.shop
├── domain/
│   ├── user/
│   │   ├── User.java
│   │   ├── UserRepository.java                  ← port (CUD + 단순 조회)
│   │   └── UserQueryRepository.java             ← port (복잡 검색)
│   └── ...
├── application/
│   ├── user/
│   │   ├── SignupUseCase.java
│   │   └── SearchUsersUseCase.java
│   └── ...
└── infrastructure/
    └── persistence/
        ├── jpa/user/
        │   ├── UserJpaEntity.java
        │   ├── UserJpaRepository.java           (Spring Data interface)
        │   └── JpaUserRepository.java           ← UserRepository 구현
        └── mybatis/user/
            ├── UserQueryMapper.java
            ├── UserSearchCriteria.java
            ├── UserSearchResult.java
            ├── MyBatisUserQueryRepository.java  ← UserQueryRepository 구현
            └── UserQueryMapper.xml              (resources/mapper/user/)
```

→ JPA / MyBatis 가 완전히 격리된 패키지. **도메인은 둘 다 모름**.

---

## 9.5 Recipe 별 JPA + MyBatis 분담 패턴

[[../api-design/api-design#0.5 ORM 정책]] 의 **공존 모드**. CUD 는 JPA, 복잡 read 는 MyBatis. Port 를 2개로 분리.

### 9.5.1 [[../api-design/signup|signup]] — 단순 CUD 만 (MyBatis 불필요)

- `UserRepository` = **JPA only** ([[jpa#11.5.1]])
- 검색 / 통계가 안 들어가므로 MyBatis 추가 이유 X.

### 9.5.2 [[../api-design/login-jwt|login-jwt]] — refresh token 단순 CUD (JPA only)

- `RefreshTokenRepository` = **JPA only**
- 단, 토큰 만료 통계 / 디바이스별 활성 토큰 보고서 같은 화면이 생기면 별도 `RefreshTokenQueryRepository` (MyBatis) 추가

### 9.5.3 [[../api-design/product-crud|product-crud]] — Aggregate CUD = JPA, 검색은 분리

```java
// domain/product/
public interface ProductRepository {                  // ✅ JPA
    Product save(Product product);
    Optional<Product> findById(ProductId id);
    Optional<Product> findByIdForUpdate(ProductId id);
}

public interface ProductQueryRepository {             // ✅ MyBatis (검색 / 보고서)
    Page<ProductSummary> search(ProductSearchCriteria c, Pageable p);
    List<TopSellingProduct> topSelling30d(int limit);
    List<RevenueByBrand> revenueByBrand(LocalDate from, LocalDate to);
}
```

→ `ProductRepository` 는 JPA Adapter 가 구현 ([[jpa#11.5.3]]). `ProductQueryRepository` 는 MyBatis Adapter 가 구현 ([[mybatis#10.5.3]]).

### 9.5.4 [[../api-design/product-search|product-search]] — **MyBatis 가 main**

검색 자체가 product-search 의 본 작업. 그래도 패턴은 유지:

```java
public interface ProductQueryRepository {
    Page<ProductSummary> search(ProductSearchCriteria c, Cursor cursor, int limit);
}
```

→ JPA Specification 으로 시작 → 트래픽 / 복잡도 증가 시 같은 port 의 구현체를 MyBatis 로 교체. Service / Controller 는 변동 없음.

### 9.5.5 [[../api-design/cart|cart]] — guest = Redis / member = JPA (MyBatis 없음)

- 외부 보고서 화면이 필요하면 → MyBatis Query Repository 추가
- 일반 운영에선 JPA + Redis 만으로 충분

### 9.5.6 [[../api-design/order-stock|order-stock]] — 트랜잭션 = JPA / 통계 / 정합성 점검 = MyBatis

```java
public interface OrderRepository {                    // ✅ JPA (트랜잭션 + 동시성)
    Order save(Order order);
    Optional<Order> findById(OrderId id);
    Optional<Order> findByIdempotencyKey(String key);
    List<Order> findExpiredCandidates(Instant before, int limit);
}

public interface OrderQueryRepository {               // ✅ MyBatis
    Page<OrderRow> searchAdmin(OrderSearchCriteria c, Pageable p);
    List<DailyOrderStats> daily(LocalDate from, LocalDate to);
    List<Order> findUnreconciled();                   // PG 정합성 점검
}
```

핵심:
- **재고 차감 같은 정확성 critical 변경은 JPA** — `@Version`, `@Lock`, dirty checking 안전
- **운영 화면 / 통계는 MyBatis** — 복잡 JOIN + GROUP BY 자유

### 9.5.7 [[../api-design/payment-pg|payment-pg]] — 결제 = JPA / webhook 멱등 = MyBatis (선택)

```java
public interface PaymentRepository {                  // ✅ JPA
    Payment save(Payment p);
    Optional<Payment> findById(PaymentId id);
    Optional<Payment> findByOrderId(OrderId orderId);
    Optional<Payment> findByPgPaymentKey(String key);
}

public interface PaymentWebhookRepository {           // ✅ MyBatis (INSERT ON CONFLICT)
    boolean tryRecord(WebhookRow row);                // 멱등 — true = 새로 받음, false = 이미 있음
    void markProcessed(String eventId, String result, Instant at);
    List<WebhookRow> findUnprocessed(int limit);
}

public interface PaymentQueryRepository {             // ✅ MyBatis (정산)
    DailySettlement dailySettlement(LocalDate date);
    List<PaymentReconciliationGap> drifts();          // PG vs DB
}
```

MyBatis 가 빛나는 자리:
- `INSERT ... ON CONFLICT DO NOTHING` (PG) — webhook 멱등을 SQL 한 줄
- 결제 / 환불 / 정산 보고서

---

## 9.6 어떤 레시피에 어떤 분담 — 한눈에 보는 표

| Recipe | JPA 담당 | MyBatis 담당 | 비고 |
| --- | --- | --- | --- |
| signup | `UserRepository` | — | 단순 CUD |
| login-jwt | `RefreshTokenRepository` | — | 단순 CUD |
| password-reset | `PasswordResetTokenRepository` | — | 단순 CUD |
| product-crud | `ProductRepository` (Aggregate) | — | 자식 컬렉션 cascade 자동 |
| product-search | `ProductRepository` (CUD) | `ProductQueryRepository` (검색) | **MyBatis 가 main** |
| cart | `CartRepository` (member) | — | guest 는 Redis |
| order-stock | `OrderRepository` (트랜잭션) | `OrderQueryRepository` (통계 / 정합성) | 정확성 vs 복잡 read 분리 |
| payment-pg | `PaymentRepository` | `PaymentWebhookRepository`, `PaymentQueryRepository` | webhook 멱등 = ON CONFLICT |

규칙:
- 도메인의 **CUD / 트랜잭션 / 동시성 critical** = JPA
- **검색 / 통계 / 보고서 / 정합성 점검** = MyBatis
- 한 레시피에 두 ORM 이 같이 들어가도 **port (interface) 가 분리되어 있으면 모드 변경이 쉬움**

---

## 10. 멀티 DataSource (참고)

같은 DB 가 아니라 **2 개 DB** 사용 시:

```yaml
spring:
  primary-datasource:
    url: jdbc:postgresql://primary:5432/shop
    ...
  audit-datasource:
    url: jdbc:postgresql://audit:5432/audit_log
    ...
```

이 경우 **각 DataSource 별로 TransactionManager 와 SqlSessionFactory 분리** 필요. 본 문서 범위 밖 — `multi-datasource.md` (다음).

---

## 11. 함정 모음

### 함정 1 — 같은 트랜잭션의 dirty read
JPA 변경 + MyBatis 즉시 read = 옛 값. **`em.flush()`** 또는 분리.

### 함정 2 — 같은 entity 양쪽으로 CUD
영속성 컨텍스트와 충돌. **CUD 는 JPA 단독**.

### 함정 3 — `@Transactional` 누락 (MyBatis 쪽)
SqlSession 이 매 호출마다 새로 → connection 매번 새로. **`@Transactional` 필수**.

### 함정 4 — MyBatis 매퍼 결과를 도메인 객체로 직접 매핑
도메인이 MyBatis 에 종속. **`UserRow` (DTO) 로 받고 application 또는 adapter 에서 도메인 변환**.

### 함정 5 — 매퍼 / Entity 컬럼명 불일치
`user_name` (DB) ↔ `userName` (JPA 자동) vs `user_name` (MyBatis 기본) — `map-underscore-to-camel-case = true` 로 통일.

### 함정 6 — JPA cache + MyBatis cache 동시 사용
복잡. **MyBatis cache 끄고 JPA 1차 캐시만 활용** 또는 외부 Redis 캐시.

### 함정 7 — 같은 mapper 안에서 JPA 도 import
정신적 혼동. **mapper 는 순수 MyBatis 만**.

### 함정 8 — 트랜잭션 격리 일관성
JPA `@Transactional(isolation = ...)` 적용 시 MyBatis 도 그 격리에서 동작 — OK. 단 명시적 SqlSession 사용 시 별도.

### 함정 9 — Spring Boot 2 → 3 마이그레이션
`javax.persistence.*` → `jakarta.persistence.*` (JPA), MyBatis-Spring-Boot-Starter 3.0 필요.

### 함정 10 — Spring Data JPA 의 `findById` 가 영속성 컨텍스트에 캐시
같은 트랜잭션에서 두 번째 `findById` = SQL 안 나감. MyBatis 의 변경을 다시 가져오려면 `em.refresh(entity)` 또는 새 트랜잭션.

---

## 12. 운영 체크리스트

- [ ] 단일 DataSource + 단일 TransactionManager 확인
- [ ] JPA `open-in-view: false`
- [ ] MyBatis `map-underscore-to-camel-case: true`
- [ ] CUD 는 JPA 단일 담당, MyBatis 는 read 중심
- [ ] 같은 트랜잭션 안 JPA write + MyBatis read 시 `em.flush()`
- [ ] mapper / Entity 컬럼 매핑 일치 검증 (IT)
- [ ] `@Transactional` 의 readOnly 명시 (MyBatis 의 select 만 메서드)
- [ ] 두 도구의 SQL 로그 (p6spy 가 둘 다 잡음)
- [ ] 도메인은 `UserRepository` / `UserQueryRepository` 두 port 만 알도록
- [ ] 새 매퍼 / Entity 추가 시 IT 회귀 (보고서가 CUD 깨뜨리지 않음)

---

## 13. 마이그레이션 전략

### A → JPA-only 인데 MyBatis 추가하고 싶을 때

1. 복잡 검색 / 통계 endpoint 가 JPQL 로 표현이 어려운가 확인
2. Yes 면 그 영역만 MyBatis Query Repository 분리
3. mapper / DTO / typeHandler 추가
4. domain layer 의 port 분리 (CUD 와 Query 둘로)

### B → MyBatis-only 인데 JPA 도입

1. **새 도메인** 부터 JPA 사용
2. 기존 mapper 는 그대로 유지 (마이그레이션 X)
3. 점진적으로 mapper → JPA 전환 (Aggregate 단위로)

### C → JPA + MyBatis 의 deprecation

오랜 운영 후 정리 시:
- JPA 가 부족했던 영역 = `JOOQ` 또는 `QueryDSL` 로 이전 검토
- MyBatis 가 부족했던 영역 = JPA Specification

---

## 14. 관련

- [[database|↑ database hub]]
- [[jpa]] — JPA 본편
- [[mybatis]] — MyBatis 본편
- [[../api-design/api-design|↗ API 레시피]] — port 분리 패턴
- [[../pitfalls/transaction-pitfalls]] (예정)
