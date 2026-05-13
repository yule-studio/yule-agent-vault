---
title: "Spring Data JPA / Hibernate — 본편"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T15:45:00+09:00
tags:
  - backend
  - java-spring
  - database
  - jpa
  - hibernate
---

# Spring Data JPA / Hibernate — 본편

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 영속성 컨텍스트 / Repository / fetch / OSIV / 함정 |

**[[database|↑ database hub]]**

> 가장 흔한 사고는 [[../pitfalls/n-plus-one]] 와 [[../pitfalls/transaction-pitfalls]].

---

## 1. JPA 와 Hibernate / Spring Data JPA

```
JPA (jakarta.persistence)  ← 표준 인터페이스 (스펙)
   │
   ├ Hibernate              ← 구현체 (Spring Boot 기본)
   │   └ Spring Data JPA    ← Repository 추상화 + 자동 구현
   │
   ├ EclipseLink, DataNucleus  (덜 쓰임)
```

코드 99% 는 `jakarta.persistence.*` + `org.springframework.data.jpa.*` 만 사용. Hibernate 내부 API 는 lazy proxy / hint / type 에서만.

---

## 2. 표준 의존성 (이미 build.gradle.kts 에 있음)

```kotlin
implementation("org.springframework.boot:spring-boot-starter-data-jpa")
runtimeOnly("org.postgresql:postgresql")
implementation("org.flywaydb:flyway-core")
implementation("org.flywaydb:flyway-database-postgresql")
// dev/staging 만
implementation("com.github.gavlyukovskiy:p6spy-spring-boot-starter:1.9.0")
```

```yaml
spring:
  jpa:
    open-in-view: false                  # ⚠️ 무조건 false. 이유는 §10.
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        format_sql: true
        default_batch_fetch_size: 100    # N+1 안전망
        jdbc.batch_size: 30
        order_inserts: true
        order_updates: true
        generate_statistics: true        # dev/staging
        query.in_clause_parameter_padding: true   # PG plan cache 친화
```

---

## 3. Entity — 매핑 기본

```java
// src/main/java/com/example/shop/infrastructure/persistence/jpa/user/UserJpaEntity.java
@Entity
@Table(name = "users")
public class UserJpaEntity {

    @Id
    @Column(length = 26)
    private String id;                           // ULID — 직접 발급 (auto X)

    @Column(nullable = false, length = 254, unique = true)
    private String email;

    @Column(name = "password_hash", nullable = false, length = 255)
    private String passwordHash;

    @Column(nullable = false, length = 100)
    private String name;

    @Enumerated(EnumType.STRING)                 // ⚠️ ORDINAL X — enum 순서 변경 시 사고
    @Column(nullable = false, length = 30)
    private UserStatus status;

    @Version
    @Column(nullable = false)
    private long version;                        // 낙관 락

    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;

    @Column(name = "updated_at", nullable = false)
    private Instant updatedAt;

    protected UserJpaEntity() {}                 // JPA 가 reflection 으로 호출
    public UserJpaEntity(/* ... */) { /* ... */ }

    // getters / package-private setters
}
```

### 3.1 ID 전략

| 전략 | 어노테이션 | 비고 |
| --- | --- | --- |
| AUTO_INCREMENT | `@GeneratedValue(strategy = GenerationType.IDENTITY)` | MySQL / PG SERIAL |
| Sequence | `@GeneratedValue(strategy = SEQUENCE, generator = "...")` | PG / Oracle |
| UUID | `@UuidGenerator` (Hibernate 6.2+) | 추측 불가 |
| **ULID / 직접 발급** | `@Id` + 어플리케이션에서 ID 부여 | **권장** — 시간순 + 추측 불가 + DB 독립 |

```java
// 도메인 layer 에서 ID 생성
public class User {
    public static User register(UserId id, ...) { ... }      // id 를 인자로 받음
}

// Application service
var user = User.register(new UserId(idGenerator.next()), ...);
```

→ DB IDENTITY 의존 X → 새로 만든 entity 도 ID 가 즉시 있음. Aggregate root 다루기 깔끔.

### 3.2 enum / JSON / Embedded

```java
@Enumerated(EnumType.STRING)                     // ENUM 컬럼이 아닌 VARCHAR
private OrderStatus status;

@JdbcTypeCode(SqlTypes.JSON)                     // Hibernate 6.2+
@Column(columnDefinition = "jsonb")
private Map<String, Object> metadata;

@Embedded
private ShippingAddress shipping;                // ShippingAddress 의 필드를 같은 테이블에

@AttributeOverrides({...})                       // embedded 필드명 / 컬럼 매핑
```

### 3.3 컬렉션 / 관계

```java
@OneToMany(mappedBy = "product",
           cascade = CascadeType.ALL,            // 부모 저장 시 자식도
           orphanRemoval = true,                 // remove(item) → DELETE
           fetch = FetchType.LAZY)               // ⚠️ EAGER 금지
@BatchSize(size = 100)                           // N+1 안전망
private final List<ProductOptionJpaEntity> options = new ArrayList<>();

@ManyToOne(fetch = FetchType.LAZY)               // 단일도 LAZY 안전
@JoinColumn(name = "brand_id", nullable = false)
private BrandJpaEntity brand;
```

> **함정**: `@OneToMany` 의 `cascade = ALL + orphanRemoval = true` 가 안 되면 자식 변경이 DB 미반영. 도메인 ↔ JPA 분리 패턴에서는 [[../api-design/product-crud#6.5]] 처럼 직접 매핑.

---

## 4. Repository — Spring Data JPA

```java
// src/main/java/com/example/shop/infrastructure/persistence/jpa/user/UserJpaRepository.java
public interface UserJpaRepository
    extends JpaRepository<UserJpaEntity, String>,
            JpaSpecificationExecutor<UserJpaEntity> {

    // 메서드 이름 → 쿼리 자동 생성
    Optional<UserJpaEntity> findByEmail(String email);
    boolean existsByEmail(String email);

    // 명시 JPQL
    @Query("from UserJpaEntity u where lower(u.email) = lower(:email)")
    Optional<UserJpaEntity> findByEmailIgnoreCase(@Param("email") String email);

    // Native SQL (PG 함수 등)
    @Query(value = "SELECT * FROM users WHERE lower(email) = lower(:email)", nativeQuery = true)
    Optional<UserJpaEntity> findByEmailNative(@Param("email") String email);

    // EntityGraph — N+1 회피
    @EntityGraph(attributePaths = {"roles", "profile"})
    @Override
    Optional<UserJpaEntity> findById(String id);

    // 비관 락
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select u from UserJpaEntity u where u.id = :id")
    Optional<UserJpaEntity> findByIdForUpdate(@Param("id") String id);

    // 일괄 update
    @Modifying
    @Query("update UserJpaEntity u set u.status = :status where u.id = :id")
    int updateStatus(@Param("id") String id, @Param("status") UserStatus status);
}
```

### 4.1 메서드 명명 규칙

| 키워드 | SQL |
| --- | --- |
| `findBy<X>` | SELECT WHERE X |
| `findAllBy<X>OrderBy<Y>` | SELECT WHERE X ORDER BY Y |
| `existsBy<X>` | SELECT 1 |
| `countBy<X>` | SELECT count(*) |
| `deleteBy<X>` (`@Modifying`) | DELETE WHERE X |
| `findBy<X>AndY` / `Or<Y>` | WHERE X AND Y / OR Y |
| `IgnoreCase`, `Like`, `In`, `Between`, `IsNull` | 표현식 |

> **함정**: 메서드 이름이 길어지면 가독성 ↓. 5개 이상 조건 합성은 `@Query` 또는 `Specification` 으로.

### 4.2 Specification — 동적 WHERE

```java
public final class UserSpecs {
    public static Specification<UserJpaEntity> isActive() {
        return (root, q, cb) -> cb.equal(root.get("status"), "ACTIVE");
    }
    public static Specification<UserJpaEntity> emailLike(String pattern) {
        if (pattern == null || pattern.isBlank()) return null;
        return (root, q, cb) -> cb.like(cb.lower(root.get("email")),
                                        "%" + pattern.toLowerCase() + "%");
    }
}

repo.findAll(Specification.where(UserSpecs.isActive())
                         .and(UserSpecs.emailLike(query)),
             PageRequest.of(0, 20));
```

→ 검색 / 필터 화면에 표준. [[../api-design/product-search]] §5.2 참고.

### 4.3 Projection / DTO

```java
// 1. Interface projection
public interface UserSummary {
    String getId();
    String getEmail();
    String getName();
}

@Query("select u.id as id, u.email as email, u.name as name from UserJpaEntity u")
List<UserSummary> findSummaries();

// 2. Class projection
@Query("""
    select new com.example.shop.UserSummaryDto(u.id, u.email, u.name)
    from UserJpaEntity u
""")
List<UserSummaryDto> findSummariesDto();

// 3. Record projection (Hibernate 6.2+)
public record UserSummaryRecord(String id, String email, String name) {}
@Query("select new com.example.shop.UserSummaryRecord(u.id, u.email, u.name) from UserJpaEntity u")
List<UserSummaryRecord> findSummariesRecord();
```

→ 목록은 DTO Projection 이 가장 빠름 (N+1 무관, 컬럼 적음).

---

## 5. 영속성 컨텍스트 (Persistence Context)

JPA 의 핵심 마법. 트랜잭션 안에서 동작:

```
@Transactional
service.update(id) {
    var u = repo.findById(id);     ← (1) DB 조회 → 영속성 컨텍스트에 저장
    u.changeName("new");           ← (2) 객체만 변경. DB 호출 X.
}                                   ← 트랜잭션 커밋 시점에 dirty checking → UPDATE 발사
```

핵심 효과:
- **dirty checking** — setter 호출만으로 자동 UPDATE
- **1차 캐시** — 같은 트랜잭션 안 같은 ID 두 번 조회 = 한 번만 SQL
- **lazy proxy** — `@ManyToOne(LAZY)` 의 객체는 처음 접근 시점에 SQL

### 5.1 dirty checking 함정

```java
@Transactional
public void changeName(UserId id, String newName) {
    var user = userRepo.findById(id.value()).orElseThrow();
    user.setName(newName);                       // OK — UPDATE 발사됨
    // userRepo.save(user) 호출 안 해도 됨
}
```

```java
public void changeName(UserId id, String newName) {     // ⚠️ @Transactional 없음
    var user = userRepo.findById(id.value()).orElseThrow();
    user.setName(newName);                       // detached state — 변경 안 됨
}
```

→ **`@Transactional` 없으면 dirty checking 동작 X**.

### 5.2 `save` 의 의미

```java
repo.save(entity);
```

- 영속화: 새로 만든 entity 면 INSERT
- 병합: detached entity (다른 트랜잭션에서 가져온) 면 MERGE → 새 객체 반환
- 영속 상태: 이미 영속이면 no-op (dirty checking 으로 처리)

→ `save` 반환값을 사용해야 안전 (`var saved = repo.save(e); use(saved);`).

### 5.3 `flush` / `clear`

```java
em.flush();    // 변경 사항을 DB 에 SQL 발사 (아직 commit 아님)
em.clear();    // 영속성 컨텍스트 비움 (객체들이 detached 됨)
```

대량 처리에서:

```java
@Transactional
public void importMany(List<UserData> data) {
    for (int i = 0; i < data.size(); i++) {
        var entity = new UserJpaEntity(...);
        em.persist(entity);
        if (i % 50 == 0) {           // 50개마다
            em.flush();              // SQL 발사
            em.clear();              // 메모리 비움
        }
    }
}
```

→ OOM 방지.

---

## 6. 트랜잭션

```java
@Transactional                       // 기본 — REQUIRED (전파)
public void update(...) { ... }

@Transactional(readOnly = true)      // 읽기 — Hibernate dirty checking 끔
public Detail get(...) { ... }

@Transactional(propagation = Propagation.REQUIRES_NEW)    // 새 트랜잭션 강제
public void logAudit(...) { ... }

@Transactional(isolation = Isolation.REPEATABLE_READ)
public void delicate(...) { ... }
```

### 6.1 readOnly 의 효과

- Hibernate `FlushMode.MANUAL` — dirty checking 안 함 → 성능 ↑
- 일부 routing datasource 에서 read replica 로 보냄
- 단순 read-heavy 메서드에 항상

### 6.2 self-invocation 함정 ([[../pitfalls/transaction-pitfalls]] 깊이)

```java
class Service {
    public void a() {
        b();                          // ⚠️ 같은 클래스 내부 호출
    }
    @Transactional
    public void b() { ... }           // a() 가 호출하면 트랜잭션 적용 X (AOP proxy 미작동)
}
```

해결:
1. 다른 빈으로 분리
2. AspectJ weaving (복잡)
3. self-injection (`@Autowired private Service self; self.b();`)

→ 보통 1번.

---

## 7. Fetch 전략

기본:
- `@ManyToOne` / `@OneToOne` → **EAGER** (위험!)
- `@OneToMany` / `@ManyToMany` → LAZY

**규칙: 모두 LAZY 로**:

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "brand_id")
private BrandJpaEntity brand;
```

필요할 때 `@EntityGraph` 로 fetch.

### 7.1 EntityGraph

```java
@EntityGraph(attributePaths = {"brand", "options", "images"})
Optional<ProductJpaEntity> findWithAllById(String id);

@EntityGraph(value = "Product.detail", type = EntityGraphType.LOAD)
Optional<ProductJpaEntity> findById(String id);     // overrides Spring Data
```

```java
// Entity 위에 named entity graph 정의 (재사용)
@NamedEntityGraph(name = "Product.detail", attributeNodes = {
    @NamedAttributeNode("brand"),
    @NamedAttributeNode("options"),
    @NamedAttributeNode("images")
})
@Entity
class ProductJpaEntity { ... }
```

### 7.2 JOIN FETCH

```java
@Query("select p from ProductJpaEntity p join fetch p.brand where p.id = :id")
Optional<ProductJpaEntity> bySingle(@Param("id") String id);
```

→ 단일 컬렉션 fetch 만. 두 컬렉션 fetch = cartesian. **`@EntityGraph` 권장**.

---

## 8. 낙관 락 / 비관 락

### 8.1 낙관 락 (`@Version`)

```java
@Entity
class ProductJpaEntity {
    @Version
    @Column(nullable = false)
    private long version;
}

// UPDATE 시 자동
// UPDATE product SET ..., version = ? WHERE id = ? AND version = ?
// rowCount 0 → OptimisticLockingFailureException
```

retry:

```java
@Retryable(
    value = OptimisticLockingFailureException.class,
    maxAttempts = 5,
    backoff = @Backoff(delay = 10, multiplier = 2)
)
@Transactional
public void update(...) { ... }
```

### 8.2 비관 락 (`SELECT FOR UPDATE`)

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000"))
@Query("select p from ProductJpaEntity p where p.id = :id")
Optional<ProductJpaEntity> findByIdForUpdate(@Param("id") String id);
```

→ [[../api-design/order-stock#3.3]] 깊이 적용.

---

## 9. Batch / Performance

### 9.1 batch insert

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc.batch_size: 30
        order_inserts: true
        order_updates: true
```

```java
@Transactional
public void importMany(List<X> rows) {
    for (var i = 0; i < rows.size(); i++) {
        em.persist(new XJpaEntity(rows.get(i)));
        if (i % 30 == 0) {
            em.flush();
            em.clear();
        }
    }
}
```

### 9.2 가급적 `saveAll` 사용

```java
repo.saveAll(entities);      // 내부적으로 batch
```

> **함정**: `@GeneratedValue(IDENTITY)` 는 batch insert 불가 (각 INSERT 후 ID 받아야). **직접 ID 발급 (ULID)** 또는 SEQUENCE.

### 9.3 DTO Projection 으로 컬럼 줄이기 (앞 §4.3)

### 9.4 `@Query countQuery` 분리

```java
@Query(value = "select p from ProductJpaEntity p where p.status = :s",
       countQuery = "select count(p) from ProductJpaEntity p where p.status = :s")
Page<ProductJpaEntity> findByStatus(@Param("s") String s, Pageable pageable);
```

---

## 10. OSIV (Open Session In View) — 반드시 끄기

```yaml
spring:
  jpa:
    open-in-view: false        # ⚠️ 무조건 false
```

OSIV = HTTP 요청 시작 ~ 응답 종료까지 영속성 컨텍스트 / 트랜잭션 살림.

- 장점: Controller / View 에서 lazy 접근 가능 (LazyInit 회피)
- 단점:
  - **DB connection 을 응답 끝까지 보유** → connection pool 고갈
  - 트랜잭션 경계가 흐려짐 → 어디서 무슨 SQL 나가는지 불명확
  - 응답 직전 lazy 접근 = N+1 폭발

**규칙**: OSIV 끄고, Service 의 `@Transactional` 안에서 필요한 모든 데이터 fetch 후 DTO 변환. View / Controller 에서는 detached.

---

## 11. Repository Adapter 패턴 — 도메인 격리

도메인이 JPA 에 종속되지 않도록 ([[../api-design/signup#6.5]] 참고):

```java
// 도메인 port
public interface UserRepository {
    Optional<User> findByEmail(Email email);
    User save(User user);
    boolean existsByEmail(Email email);
}

// infrastructure adapter
@Repository
public class JpaUserRepositoryAdapter implements UserRepository {
    private final UserJpaRepository spring;
    private final Clock clock;

    @Override public Optional<User> findByEmail(Email email) {
        return spring.findByEmailIgnoreCase(email.value()).map(this::toDomain);
    }
    @Override public User save(User user) { ... }

    private User toDomain(UserJpaEntity e) {
        return User.reconstitute(...);
    }
}
```

> 도메인 클래스에 `@Entity` 안 붙임. 두 클래스 (User / UserJpaEntity) 가 mirror.
> 작은 프로젝트는 합쳐도 OK. 큰 프로젝트는 분리 권장.

---

## 12. 함정 모음

### 함정 1 — `open-in-view: true`
기본값 (안 끄면 true). 모든 사고의 원흉. **반드시 false**.

### 함정 2 — `@ManyToOne` 기본 EAGER
LAZY 명시 안 하면 매번 JOIN. **명시적 LAZY**.

### 함정 3 — N+1
[[../pitfalls/n-plus-one]] 본편 참고. `@EntityGraph` + `default_batch_fetch_size`.

### 함정 4 — `EnumType.ORDINAL`
enum 순서 바뀌면 모든 row 의 의미 바뀜. **`STRING`**.

### 함정 5 — `Cascade.REMOVE` 무지성
부모 삭제 = 자식 다 삭제. 의도와 다를 수 있음. **`PERSIST` / `MERGE` 정도만**.

### 함정 6 — `@OneToMany` 둘 다 fetch join
cartesian product. **EntityGraph 사용**.

### 함정 7 — `@Modifying` 후 영속성 컨텍스트 불일치
`@Modifying @Query("update ...")` 가 발사한 UPDATE 는 영속성 컨텍스트 모름. **`@Modifying(clearAutomatically = true, flushAutomatically = true)`** 또는 직접 `em.flush()`, `em.clear()`.

### 함정 8 — `save(entity)` 호출 무지성
영속 상태 entity 는 `save` 안 해도 됨. detached / new 만 `save`. **dirty checking 신뢰**.

### 함정 9 — `findById` 후 즉시 `em.detach`
1차 캐시 / dirty checking 못 씀. 의도 없으면 안 함.

### 함정 10 — `@Transactional` 메서드를 같은 클래스에서 호출
proxy 미작동. self-invocation. **다른 빈으로 분리**.

### 함정 11 — Hibernate batch insert 시 `IDENTITY` 사용
batch 불가. **ULID 직접 발급** 또는 sequence.

### 함정 12 — `Pageable` + JOIN FETCH
Hibernate 가 메모리에서 paginate (HHH000104). **`@EntityGraph` + Pageable**.

### 함정 13 — `equals` / `hashCode` 미오버라이드
Set / Map 사용 시 같은 ID 의 다른 인스턴스가 다른 객체로 취급. **ID 기반 equals**.

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof UserJpaEntity that)) return false;
    return id != null && id.equals(that.id);
}
@Override
public int hashCode() { return Objects.hashCode(id); }   // 또는 getClass().hashCode()
```

### 함정 14 — `@Transactional` 의 `rollbackFor` 누락
Unchecked exception (RuntimeException) 만 자동 rollback. **Checked exception 은 명시**:

```java
@Transactional(rollbackFor = Exception.class)
```

또는 도메인 예외를 RuntimeException 으로.

---

## 13. 운영 체크리스트

- [ ] `open-in-view: false`
- [ ] `ddl-auto: validate` + Flyway
- [ ] `default_batch_fetch_size = 100`
- [ ] `EnumType.STRING`
- [ ] 모든 `@OneToMany` `LAZY` + `@BatchSize` 또는 `@EntityGraph`
- [ ] `@Version` 으로 낙관 락 (변경 빈도 높은 entity)
- [ ] `@Transactional` 의 `readOnly` 명시
- [ ] `@Modifying` 후 `clearAutomatically = true`
- [ ] dev/staging `p6spy` + `generate_statistics`
- [ ] IT 에 N+1 / 쿼리 수 assertion

---

## 14. 관련

- [[database|↑ database hub]]
- [[mybatis]] — MyBatis 비교
- [[jpa-mybatis-coexist]] — 같이 쓸 때
- [[../pitfalls/n-plus-one]] · [[../pitfalls/transaction-pitfalls]]
- [[../api-design/signup#6.5]] · [[../api-design/product-crud#5.8]] — Adapter 적용 예
