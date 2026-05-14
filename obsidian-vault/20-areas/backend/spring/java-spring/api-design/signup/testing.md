---
title: "signup §8 — 테스트 시나리오 / 단위 / 통합"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:48:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - signup
  - testing
---

# signup §8 — 테스트 시나리오 / 단위 / 통합

**[[signup|↑ signup hub]]**  ·  ← [[transactions]]  ·  → [[operations]]

---

## 1. 테스트 시나리오 표

[[requirements#2 완료 조건]] 의 항목별 매핑.

| # | 시나리오 | 입력 | 기대 결과 | 종류 | 우선순위 |
| --- | --- | --- | --- | --- | --- |
| 1 | 정상 가입 | 유효한 email + 강한 password + 약관 동의 | 200, status=PENDING_VERIFICATION, DB row 1, outbox row 1 | 통합 | 🔴 必 |
| 2 | 약한 password (8자 미만) | password="short" | 422, BADREQ_002, DB 변동 X | 단위 + 통합 | 🔴 必 |
| 3 | 잘못된 email 형식 | email="not-an-email" | 422, BADREQ_002 (Bean Validation) | 단위 (Controller) | 🔴 必 |
| 4 | 약관 미동의 | termsAgreed=false | 400, AssertTrue 위반 | 단위 (Controller) | 🔴 必 |
| 5 | 중복 이메일 | 이미 가입된 email 다시 | 409, BADREQ_004, 새 row X | 통합 | 🔴 必 |
| 6 | 대소문자 다른 이메일 | "Alice@X.com" 후 "alice@x.com" | 409 (case-insensitive UNIQUE) | 통합 | 🔴 必 |
| 7 | 도메인 이벤트 발행 확인 | 정상 가입 | `UserRegistered` event 1번 publish | 단위 (UseCase mock) | 🟡 권장 |
| 8 | 동시성 race | 같은 email 2개 동시 요청 | 정확히 1개만 성공, 1개는 409 | 통합 (멀티스레드) | 🟡 권장 |
| 9 | 평문 password 노출 X | 정상 가입 | 응답 JSON 에 password 없음 | 통합 (응답 검증) | 🔴 必 |
| 10 | toString 마스킹 | request 로깅 | `password=***` | 단위 | 🟢 |
| 11 | 약관 history 저장 | 정상 가입 | `user_terms_consent_history` row 추가 (필수 + 선택) | 통합 | 🟡 권장 |
| 12 | outbox AFTER_COMMIT | 정상 가입 | `email_outbox` 에 row, 단 트랜잭션 rollback 시 적재 X | 통합 | 🟡 권장 |
| 13 | Idempotency-Key 재사용 | 같은 key 2번 호출 | 둘 다 200, userId 동일 | 통합 | 🟡 권장 (옵션) |

---

## 2. 단위 테스트 — Domain

```java
// src/test/java/com/example/shop/domain/user/UserTest.java
class UserTest {

    @Test
    void register_raises_UserRegistered_event() {
        var user = User.register(
            new UserId("01HZ" + "X".repeat(22)),
            new Email("a@b.com"),
            new PasswordHash("$argon2id$v=19$m=65536,t=3,p=4$x$y"),
            "alice",
            Instant.parse("2026-05-14T00:00:00Z")
        );
        assertThat(user.status()).isEqualTo(UserStatus.PENDING_VERIFICATION);
        var events = user.pullDomainEvents();
        assertThat(events).hasSize(1).first().isInstanceOf(UserRegistered.class);
    }

    @Test
    void pullDomainEvents_clears_buffer() {
        var u = newPendingUser();
        assertThat(u.pullDomainEvents()).hasSize(1);
        assertThat(u.pullDomainEvents()).isEmpty();   // 두 번째 pull = 비어있음
    }

    @Test
    void verifyEmail_moves_to_ACTIVE_only_when_PENDING() {
        var u = newPendingUser();
        u.verifyEmail();
        assertThat(u.status()).isEqualTo(UserStatus.ACTIVE);
        assertThatThrownBy(u::verifyEmail).isInstanceOf(IllegalStateException.class);
    }

    @Test
    void invalid_email_rejected() {
        assertThatThrownBy(() -> new Email("not-an-email"))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void non_argon2_hash_rejected() {
        assertThatThrownBy(() -> new PasswordHash("$2a$10$bcrypt"))
            .isInstanceOf(IllegalArgumentException.class);
    }
}
```

---

## 3. 단위 테스트 — Application UseCase (Mockito)

```java
// src/test/java/com/example/shop/application/user/SignupUseCaseTest.java
@ExtendWith(MockitoExtension.class)
class SignupUseCaseTest {

    @Mock UserRepository users;
    @Mock PasswordEncoder encoder;
    @Mock PasswordPolicy policy;
    @Mock UserTermsConsentService termsConsent;
    @Mock IdGenerator ids;
    @Mock ApplicationEventPublisher events;

    Clock clock = Clock.fixed(Instant.parse("2026-05-14T00:00:00Z"), ZoneOffset.UTC);
    SignupUseCase sut;

    @BeforeEach
    void setup() {
        sut = new SignupUseCase(users, encoder, policy, termsConsent, ids, clock, events);
    }

    @Test
    void signs_up_new_user() {
        when(ids.next()).thenReturn("01HZ" + "X".repeat(22));
        when(users.existsByEmail(new Email("a@b.com"))).thenReturn(false);
        when(encoder.encode("Tr0ub4dor!12")).thenReturn("$argon2id$x");
        when(users.save(any())).thenAnswer(inv -> inv.getArgument(0));

        var user = sut.handle(new SignupCommand("a@b.com", "Tr0ub4dor!12", "alice", true, false));

        assertThat(user.email().value()).isEqualTo("a@b.com");
        assertThat(user.status()).isEqualTo(UserStatus.PENDING_VERIFICATION);
        verify(events).publishEvent(any(UserRegistered.class));
        verify(termsConsent).save(any(), eq(true), eq(false), any());
    }

    @Test
    void rejects_when_terms_not_agreed() {
        assertThatThrownBy(() ->
            sut.handle(new SignupCommand("a@b.com", "Tr0ub4dor!12", "alice", false, false))
        ).isInstanceOf(BusinessException.class)
         .hasMessageContaining("약관 동의");
        verifyNoInteractions(users, encoder);
    }

    @Test
    void rejects_duplicate_email() {
        when(users.existsByEmail(new Email("a@b.com"))).thenReturn(true);
        assertThatThrownBy(() ->
            sut.handle(new SignupCommand("a@b.com", "Tr0ub4dor!12", "alice", true, false))
        ).isInstanceOf(EmailAlreadyExistsException.class);
    }

    @Test
    void converts_data_integrity_violation_to_duplicate() {
        when(ids.next()).thenReturn("01HZ" + "X".repeat(22));
        when(users.existsByEmail(any())).thenReturn(false);
        when(encoder.encode(any())).thenReturn("$argon2id$x");
        when(users.save(any())).thenThrow(new DataIntegrityViolationException("unique"));

        assertThatThrownBy(() ->
            sut.handle(new SignupCommand("a@b.com", "Tr0ub4dor!12", "alice", true, false))
        ).isInstanceOf(EmailAlreadyExistsException.class);
    }

    @Test
    void normalizes_email_to_lowercase() {
        when(ids.next()).thenReturn("01HZ" + "X".repeat(22));
        when(users.existsByEmail(any())).thenReturn(false);
        when(encoder.encode(any())).thenReturn("$argon2id$x");
        when(users.save(any())).thenAnswer(inv -> inv.getArgument(0));

        sut.handle(new SignupCommand("  Alice@X.COM  ", "Tr0ub4dor!12", "alice", true, false));

        var captor = ArgumentCaptor.forClass(User.class);
        verify(users).save(captor.capture());
        assertThat(captor.getValue().email().value()).isEqualTo("alice@x.com");
    }
}
```

---

## 4. 통합 테스트 — Testcontainers + 실제 PostgreSQL

```java
// src/test/java/com/example/shop/integration/SignupApiIT.java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
@ActiveProfiles("test")
class SignupApiIT {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("shop_it").withUsername("test").withPassword("test");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry reg) {
        reg.add("spring.datasource.url", postgres::getJdbcUrl);
        reg.add("spring.datasource.username", postgres::getUsername);
        reg.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired TestRestTemplate rest;
    @Autowired UserJpaRepository userRepo;
    @Autowired EmailOutboxRepository outboxRepo;

    @Test
    void scenario_1_normal_signup_returns_200() {
        var body = Map.of(
            "email", "alice@x.com",
            "password", "Tr0ub4dor!12",
            "name", "Alice",
            "termsAgreed", true
        );
        var res = rest.postForEntity("/api/v1/auth/signup", body, Map.class);
        assertThat(res.getStatusCode().value()).isEqualTo(200);
        assertThat(res.getBody()).containsEntry("code", "OK_001");

        var data = (Map<?, ?>) res.getBody().get("result");
        assertThat(data.get("status")).isEqualTo("PENDING_VERIFICATION");
        assertThat(data.get("email")).isEqualTo("alice@x.com");

        // DB 검증
        var fromDb = userRepo.findByEmailIgnoreCase("alice@x.com").orElseThrow();
        assertThat(fromDb.getStatus()).isEqualTo(UserStatus.PENDING_VERIFICATION);
        assertThat(fromDb.getPasswordHash()).startsWith("$argon2id$");

        // Outbox 검증 (AFTER_COMMIT)
        var outboxRows = outboxRepo.findByToEmail("alice@x.com");
        assertThat(outboxRows).hasSize(1);
        assertThat(outboxRows.get(0).getTemplate()).isEqualTo("verification");
    }

    @Test
    void scenario_5_duplicate_email_returns_409() {
        var body = Map.of("email", "dup@x.com", "password", "Tr0ub4dor!12",
                          "name", "x", "termsAgreed", true);
        rest.postForEntity("/api/v1/auth/signup", body, Map.class);

        var res2 = rest.postForEntity("/api/v1/auth/signup", body, Map.class);
        assertThat(res2.getStatusCode().value()).isEqualTo(409);
        assertThat(res2.getBody()).containsEntry("code", "BADREQ_004");
    }

    @Test
    void scenario_6_case_insensitive_duplicate() {
        rest.postForEntity("/api/v1/auth/signup",
            Map.of("email", "Alice@X.com", "password", "Tr0ub4dor!12",
                   "name", "A", "termsAgreed", true), Map.class);
        var res = rest.postForEntity("/api/v1/auth/signup",
            Map.of("email", "alice@x.com", "password", "Tr0ub4dor!12",
                   "name", "B", "termsAgreed", true), Map.class);
        assertThat(res.getStatusCode().value()).isEqualTo(409);
    }

    @Test
    void scenario_9_response_does_not_contain_password() throws Exception {
        var body = Map.of("email", "secret@x.com", "password", "Tr0ub4dor!12",
                          "name", "X", "termsAgreed", true);
        var res = rest.postForEntity("/api/v1/auth/signup", body, String.class);
        assertThat(res.getBody()).doesNotContain("password")
                                  .doesNotContain("passwordHash")
                                  .doesNotContain("Tr0ub4dor");
    }

    @Test
    void scenario_8_concurrent_signup_with_same_email() throws Exception {
        var executor = Executors.newFixedThreadPool(2);
        var latch = new CountDownLatch(2);
        var successes = new AtomicInteger();
        var conflicts = new AtomicInteger();

        for (int i = 0; i < 2; i++) {
            executor.submit(() -> {
                try {
                    var res = rest.postForEntity("/api/v1/auth/signup",
                        Map.of("email", "race@x.com", "password", "Tr0ub4dor!12",
                               "name", "X", "termsAgreed", true), Map.class);
                    if (res.getStatusCode().value() == 200) successes.incrementAndGet();
                    else if (res.getStatusCode().value() == 409) conflicts.incrementAndGet();
                } finally { latch.countDown(); }
            });
        }
        latch.await(10, TimeUnit.SECONDS);
        executor.shutdown();

        assertThat(successes.get()).isEqualTo(1);
        assertThat(conflicts.get()).isEqualTo(1);
        assertThat(userRepo.findByEmailIgnoreCase("race@x.com")).isPresent();
    }
}
```

---

## 5. Controller 단위 테스트 (`@WebMvcTest`) — 옵션

도메인 / DB 무관, Controller layer 의 Bean Validation 만 빠르게.

```java
@WebMvcTest(SignupController.class)
class SignupControllerWebMvcTest {

    @Autowired MockMvc mvc;
    @MockBean SignupUseCase signupUseCase;

    @Test
    void invalid_email_returns_422() throws Exception {
        mvc.perform(post("/api/v1/auth/signup")
            .contentType(MediaType.APPLICATION_JSON)
            .content("""
                { "email": "not-an-email", "password": "Tr0ub4dor!12", "name": "x", "termsAgreed": true }
                """))
            .andExpect(status().isUnprocessableEntity())
            .andExpect(jsonPath("$.code").value("BADREQ_002"));
    }

    @Test
    void terms_not_agreed_returns_400_or_422() throws Exception {
        mvc.perform(post("/api/v1/auth/signup")
            .contentType(MediaType.APPLICATION_JSON)
            .content("""
                { "email": "a@x.com", "password": "Tr0ub4dor!12", "name": "x", "termsAgreed": false }
                """))
            .andExpect(status().isUnprocessableEntity());
    }
}
```

---

## 6. 테스트 카테고리별 권장 비중

| 종류 | 양 | 도구 | 속도 |
| --- | --- | --- | --- |
| 도메인 단위 (User / Email / ...) | 많음 (각 메서드 1+) | JUnit + AssertJ | ms |
| Application 단위 (UseCase mocking) | 핵심 흐름 + 분기 | JUnit + Mockito | ms |
| Controller 단위 (`@WebMvcTest`) | 입력 검증 위주 | MockMvc | 100ms |
| 통합 (`@SpringBootTest + Testcontainers`) | 시나리오 #1, #5, #6, #9 | Testcontainers | 5~30s |
| E2E (외부 API + 실 DB) | 없음 / 최소 | — | s ~ min |

---

## 7. 운영 — 회귀 테스트

- 새 PR 마다 위 13 시나리오 다 run
- 통합 테스트는 CI 의 다른 stage 분리 (속도)
- 신규 시나리오 추가 시 시나리오 표 (§1) 도 같이 갱신

---

## 8. 관련

- [[signup|↑ signup hub]]
- [[transactions]] — 이전 (§7)
- [[operations]] — 다음 (§9)
- [[requirements#2 완료 조건]] — 시나리오의 source
