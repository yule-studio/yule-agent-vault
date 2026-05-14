---
title: "Integration Tests — Testcontainers + DB"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T06:00:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - testing
  - integration-test
  - testcontainers
---

# Integration Tests — Testcontainers + DB

**[[testing|↑ testing hub]]**

> 진짜 PostgreSQL / Redis + Spring context. 단 비싸므로 critical path 만.

---

## 1. 왜 Integration Test 필요한가

**왜 필요한가**
- Unit test 가 mock 으로 격리 → JPA / SQL / 트랜잭션 / 인덱스 / 동시성 미검증.
- "Unit 통과인데 prod fail" 의 흔한 원인 = Integration 부재.

**안 하면 무슨 문제**
- N+1 query 의 silent 발생.
- @Transactional 의 propagation 잘못된 케이스.
- Hibernate 의 lazy loading 함정.
- DB constraint (UNIQUE / CHECK / FK) 의 race condition.

**대안과 왜 안 됨**
- E2E only → 매우 비싸 (분 단위), debug 어려움.
- Mock + Unit only → 통합 검증 X.

**트레이드오프**
- Testcontainers 컨테이너 시작 시간 (10-30초).
- CI 시간 ↑ — 그래도 prod 사고보다 싸다.

---

## 2. Testcontainers 설정

### 2.1 Maven / Gradle

```kotlin
testImplementation("org.testcontainers:postgresql:1.20.2")
testImplementation("org.testcontainers:junit-jupiter:1.20.2")
testImplementation("org.springframework.boot:spring-boot-starter-test")
```

### 2.2 Base class

```java
@Testcontainers
@SpringBootTest
@ActiveProfiles("test")
public abstract class IntegrationTestBase {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("yule_test")
        .withUsername("yule")
        .withPassword("yule")
        .withReuse(true);                              // 컨테이너 재사용 (속도 ↑)

    @DynamicPropertySource
    static void registerProps(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}
```

### 2.3 왜 `withReuse(true)`

- 매 테스트 클래스마다 컨테이너 시작 = 매 10-30초.
- reuse = 한 번 시작 후 재사용 = 전체 CI 시간 ↓.

**조건**
- `~/.testcontainers.properties` 에 `testcontainers.reuse.enable=true`.
- DB schema 는 매번 깨끗 (Flyway clean + migrate 또는 transaction rollback).

---

## 3. Repository / DB 테스트

```java
@DataJpaTest(includeFilters = @ComponentScan.Filter(...))
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class UserRepositoryIT extends IntegrationTestBase {

    @Autowired UserJpaRepository repo;

    @Test
    void save_and_load() {
        var entity = newUserEntity("alice@example.com");
        repo.save(entity);
        var loaded = repo.findById(entity.getId()).orElseThrow();
        assertThat(loaded.getEmail()).isEqualTo("alice@example.com");
    }

    @Test
    void lower_email_unique_index() {
        repo.save(newUserEntity("Alice@X.com"));

        assertThatThrownBy(() -> {
            repo.save(newUserEntity("alice@x.com"));   // 같은 소문자
            repo.flush();
        }).isInstanceOf(DataIntegrityViolationException.class);
    }

    @Test
    void check_constraint_local_requires_password() {
        var entity = newUserEntity("bob@example.com");
        entity.setProviderType(SocialProviderType.LOCAL);
        entity.setPasswordHash(null);

        assertThatThrownBy(() -> {
            repo.save(entity);
            repo.flush();
        }).isInstanceOf(DataIntegrityViolationException.class);
    }
}
```

### 3.1 왜 `@DataJpaTest` (`@SpringBootTest` 가 아님)

- JPA 만 — Spring context 가벼움 + 빠름.
- 트랜잭션 자동 rollback.

### 3.2 왜 `.flush()`

- JPA 의 deferred constraint check — flush 전엔 INSERT 안 함.
- 실제 DB 검증 시 flush 명시.

---

## 4. Controller + Service + DB — `@SpringBootTest`

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
class SignupControllerIT extends IntegrationTestBase {

    @Autowired MockMvc mockMvc;
    @Autowired UserJpaRepository userRepo;

    @Test
    void signup_success_returns_200_and_persists_user() throws Exception {
        var requestBody = """
            {
              "email": "alice@example.com",
              "password": "Password1234!",
              "name": "alice",
              "termsAgreed": true,
              "agreedTerms": ["01HTERMS01SERVICE", "01HTERMS02PRIVACY"]
            }
            """;

        mockMvc.perform(post("/api/v1/auth/signup")
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestBody))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.code").value("OK_001"))
            .andExpect(jsonPath("$.data.userId").isNotEmpty())
            .andExpect(jsonPath("$.data.password").doesNotExist());      // 평문 노출 X

        // DB 검증
        var users = userRepo.findAll();
        assertThat(users).hasSize(1);
        assertThat(users.get(0).getEmail()).isEqualTo("alice@example.com");
        assertThat(users.get(0).getStatus()).isEqualTo(UserStatus.PENDING_VERIFICATION);
    }

    @Test
    void duplicate_email_returns_409() throws Exception {
        userRepo.save(newUserEntity("alice@example.com"));

        mockMvc.perform(post("/api/v1/auth/signup")
                .contentType(MediaType.APPLICATION_JSON)
                .content(duplicateBody()))
            .andExpect(status().isConflict())
            .andExpect(jsonPath("$.code").value("BADREQ_004"));
    }

    @Test
    void weak_password_returns_422() throws Exception {
        mockMvc.perform(post("/api/v1/auth/signup")
                .contentType(MediaType.APPLICATION_JSON)
                .content(weakPasswordBody()))
            .andExpect(status().isUnprocessableEntity())
            .andExpect(jsonPath("$.code").value("BADREQ_002"));
    }
}
```

### 4.1 왜 `MockMvc` (REST Assured 안 씀)

- Spring 통합 — application context 시작.
- HTTP 시뮬레이션 (실 network X) — 빠름.
- REST Assured 는 E2E 용.

### 4.2 왜 `RANDOM_PORT`

- 병렬 테스트 시 포트 충돌 회피.

### 4.3 왜 `password.doesNotExist()` 검증

- 응답에 평문 password 노출 차단 검증.
- security 검증의 일부.

---

## 5. Outbox 패턴 검증 — `@TransactionalEventListener`

```java
@SpringBootTest
class OutboxIT extends IntegrationTestBase {

    @Autowired SignupUseCase useCase;
    @Autowired EmailOutboxRepository outbox;

    @Test
    void outbox_row_created_after_commit() {
        useCase.execute(validCommand("alice@example.com"));

        // 트랜잭션 commit 후 listener 가 outbox row INSERT
        await().atMost(Duration.ofSeconds(2)).untilAsserted(() -> {
            assertThat(outbox.findByToEmail("alice@example.com")).isNotEmpty();
        });
    }

    @Test
    void outbox_not_created_when_rollback() {
        assertThatThrownBy(() -> useCase.execute(invalidCommand()))
            .isInstanceOf(BusinessException.class);

        assertThat(outbox.findAll()).isEmpty();
    }
}
```

### 5.1 왜 `await()` (Awaitility)

- AFTER_COMMIT listener 가 비동기.
- 즉시 assert 시 race condition.

---

## 6. 동시성 테스트

```java
@Test
void concurrent_signup_same_email_only_one_succeeds() throws Exception {
    var executor = Executors.newFixedThreadPool(10);
    var successCount = new AtomicInteger();
    var failureCount = new AtomicInteger();
    var latch = new CountDownLatch(1);

    var futures = IntStream.range(0, 10)
        .mapToObj(i -> CompletableFuture.runAsync(() -> {
            try {
                latch.await();
                useCase.execute(commandFor("alice@example.com"));
                successCount.incrementAndGet();
            } catch (EmailAlreadyExistsException e) {
                failureCount.incrementAndGet();
            } catch (Exception e) {
                fail("unexpected: " + e);
            }
        }, executor))
        .toList();

    latch.countDown();           // 동시 시작
    futures.forEach(CompletableFuture::join);

    assertThat(successCount.get()).isEqualTo(1);     // 정확히 1개만 성공
    assertThat(failureCount.get()).isEqualTo(9);
    assertThat(userRepo.findAll()).hasSize(1);
}
```

### 6.1 왜 동시성 테스트

- DB UNIQUE constraint 의 race condition 검증.
- application 의 `existsByEmail` 만으론 부족 증명.

### 6.2 왜 `latch.countDown()`

- 모든 thread 가 같은 시점에 시작 = 진짜 race.

---

## 7. Redis Integration

```java
@Testcontainers
@SpringBootTest
class PhoneVerificationIT extends IntegrationTestBase {

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);

    @DynamicPropertySource
    static void redisProps(DynamicPropertyRegistry registry) {
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add("spring.data.redis.port", () -> redis.getMappedPort(6379));
    }

    @Autowired PhoneVerificationService service;

    @Test
    void code_expires_after_3min() throws InterruptedException {
        service.sendCode("01012345678");
        Thread.sleep(3_100);

        assertThatThrownBy(() -> service.verifyCode("01012345678", "123456"))
            .isInstanceOf(VerificationExpiredException.class);
    }
}
```

---

## 8. 데이터 cleanup

### 8.1 옵션 A: 트랜잭션 rollback (default)

```java
@SpringBootTest
@Transactional                  // 매 테스트 rollback
class FooIT { ... }
```

**왜 안 됨 (가끔)**
- AFTER_COMMIT listener — commit 안 되면 listener 발동 X.
- 비동기 / @Async — main 트랜잭션과 별도.

### 8.2 옵션 B: Flyway clean + migrate

```java
@BeforeEach
void clean() {
    flyway.clean();
    flyway.migrate();
}
```

**왜 가끔 필요**
- AFTER_COMMIT 검증 시.
- 매 테스트마다 깨끗한 상태.
- 단 시간 ↑ (수 초).

### 8.3 옵션 C: `@Sql` (selective)

```java
@Test
@Sql("/test-data/users.sql")
void test_with_fixed_data() { ... }
```

---

## 9. 함정 모음

### 함정 1 — H2 in-memory 사용
PG-specific 기능 미검증 → prod 사고.
→ Testcontainers PG.

### 함정 2 — 컨테이너 매번 시작
CI 시간 ↑.
→ withReuse(true).

### 함정 3 — `@Transactional` 의 rollback 의존
AFTER_COMMIT 검증 X.
→ Flyway clean 또는 별도 패턴.

### 함정 4 — JPA `flush()` 누락
deferred constraint check 미검증.
→ 명시 flush.

### 함정 5 — 동시성 테스트 안 함
race condition 의 silent.
→ 동시성 테스트.

### 함정 6 — Spring context 매 클래스마다 시작
시간 ↑.
→ context cache (default).

### 함정 7 — 외부 API 진짜 호출
SES / SMS 비용 + flaky.
→ Mock / WireMock.

### 함정 8 — assert 없는 테스트
"통과" 하지만 검증 X.
→ 의미 있는 assertion.

### 함정 9 — fixture 가 inline 중복
각 테스트마다 user 생성 코드.
→ Builder / common helper.

### 함정 10 — Coverage 100% 만 추구
가짜 통과 (assertion 약함).
→ mutation testing (PIT) — assertion 강도 검증.

### 함정 11 — 비동기 테스트의 sleep
flaky — 시스템 부하 따라 시간 다름.
→ Awaitility 같은 polling.

### 함정 12 — Test data 가 prod 환경에 leak
실수로 dev DB 에 testcontainer 의 URL 점.
→ profile 명시 + CI 환경 격리.

---

## 10. 관련

- [[testing|↑ testing hub]]
- [[unit-tests]]
- [[test-data-fixtures]]
- [[../database]] · [[../security/authentication-authorization]]
