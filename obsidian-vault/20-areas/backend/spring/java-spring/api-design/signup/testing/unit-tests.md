---
title: "Unit Tests — Domain / VO / Application"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:55:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - testing
  - unit-test
---

# Unit Tests — Domain / VO / Application

**[[testing|↑ testing hub]]**

> 빠른 / 격리된 테스트. **Spring context 없이** 도메인 로직 / Application service 검증.

---

## 1. 4구조 — Unit Test 의 가치

**왜 필요한가**
- 즉시 feedback — 수 ms 안에 회귀 발견.
- 도메인 로직 정확성 (invariant / 상태 전이) 검증.
- 리팩토링 시 안전망.

**안 하면 무슨 문제**
- 도메인 버그가 Integration 단계에서 발견 → 디버깅 비용 ↑.
- CI 시간 ↑ → 머지 늦어짐.
- 사용자에게 도달하는 버그 ↑.

**대안과 왜 안 됨**
- Integration only → 느림 + 도메인 격리 검증 X.
- 수동 테스트만 → 회귀 발견 어려움.

**트레이드오프**
- Mock 사용 → 진짜 동작과 차이 가능 (Integration 으로 보완).
- 작은 단위 → 시나리오 분산 (대량 테스트 코드).

---

## 2. 도메인 Aggregate — User

```java
// src/test/java/com/example/shop/domain/user/UserTest.java
class UserTest {

    @Test
    void register_raises_UserRegistered_event() {
        var user = User.register(
            new UserId(ulidString()),
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
    void cannot_changePassword_when_DELETED() {
        var u = newDeletedUser();
        assertThatThrownBy(() -> u.changePassword(new PasswordHash("$argon2id$...")))
            .isInstanceOf(IllegalStateException.class);
    }

    @Test
    void suspend_only_when_ACTIVE() {
        var u = newActiveUser();
        u.suspend("violation");
        assertThat(u.status()).isEqualTo(UserStatus.SUSPENDED);
    }

    @Test
    void register_invalid_state_throws() {
        assertThatThrownBy(() -> User.register(
            new UserId(ulidString()),
            new Email("a@b.com"),
            null,                                       // password hash 누락
            "alice",
            Instant.now()
        )).isInstanceOf(IllegalArgumentException.class);
    }
}
```

### 2.1 왜 Spring context 안 씀

- `@SpringBootTest` = 수 초 시작.
- Spring 없이 객체 직접 생성 = ms.
- 도메인은 framework 의존 X — pure java.

### 2.2 왜 AssertJ (assertEquals 아님)

- 가독성 — `assertThat(...).hasSize(1).first()...`.
- 자세한 에러 메시지.
- chaining.

### 2.3 왜 fixture builder 사용

```java
private User newPendingUser() {
    return User.register(
        new UserId(ulidString()),
        new Email("test@example.com"),
        new PasswordHash("$argon2id$v=19$m=65536,t=3,p=4$x$y"),
        "test user",
        Instant.parse("2026-05-14T00:00:00Z")
    );
}
```

- 매 테스트마다 중복 setup 회피.
- 단일 진실 — User 생성 방법 변경 시 한 곳만.

자세히: [[test-data-fixtures]].

---

## 3. Value Object 테스트

```java
class EmailTest {

    @Test
    void valid_email_accepted() {
        var email = new Email("alice@example.com");
        assertThat(email.value()).isEqualTo("alice@example.com");
    }

    @Test
    void invalid_format_rejected() {
        assertThatThrownBy(() -> new Email("not-an-email"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("invalid email");
    }

    @Test
    void null_rejected() {
        assertThatThrownBy(() -> new Email(null))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void too_long_rejected() {
        var longEmail = "a".repeat(250) + "@x.com";   // > 254
        assertThatThrownBy(() -> new Email(longEmail))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void equals_value_based() {
        assertThat(new Email("a@b.com")).isEqualTo(new Email("a@b.com"));
        assertThat(new Email("a@b.com")).isNotEqualTo(new Email("c@b.com"));
    }
}
```

### 3.1 왜 각 VO 마다 5+ tests

- 정상 / null / 형식 오류 / 길이 / 동등성.
- VO 가 도메인의 1차 검증 — 안전 critical.

---

## 4. Application Service — Mockito

```java
// src/test/java/com/example/shop/application/user/SignupUseCaseTest.java
@ExtendWith(MockitoExtension.class)
class SignupUseCaseTest {

    @Mock UserRepository users;
    @Mock UserTermsConsentRepository termsConsents;
    @Mock TermsRepository terms;
    @Mock PasswordEncoder encoder;
    @Mock PasswordPolicy passwordPolicy;
    @Mock ApplicationEventPublisher eventPublisher;
    @Mock Clock clock;
    @InjectMocks SignupUseCase useCase;

    @Test
    void signup_success() {
        // Given
        when(clock.instant()).thenReturn(Instant.parse("2026-05-14T00:00:00Z"));
        when(users.existsByEmail(any())).thenReturn(false);
        when(encoder.encode(any())).thenReturn("$argon2id$v=19$m=65536,t=3,p=4$x$y");
        when(users.save(any(User.class))).thenAnswer(inv -> inv.getArgument(0));

        // When
        var result = useCase.execute(new SignupCommand(
            "alice@example.com", "Password1234!", "alice", true, List.of("01HTERMS01")
        ));

        // Then
        assertThat(result.userId()).isNotNull();
        verify(users).save(any(User.class));
        verify(eventPublisher).publishEvent(any(UserRegistered.class));
    }

    @Test
    void duplicate_email_throws() {
        when(users.existsByEmail(any())).thenReturn(true);

        assertThatThrownBy(() -> useCase.execute(new SignupCommand(
            "alice@example.com", "Password1234!", "alice", true, List.of("01HTERMS01")
        ))).isInstanceOf(EmailAlreadyExistsException.class);

        verify(users, never()).save(any());
    }

    @Test
    void weak_password_throws() {
        doThrow(new BusinessException(ResponseCode.INVALID_INPUT_FORMAT))
            .when(passwordPolicy).validate(any(), any(), any());

        assertThatThrownBy(() -> useCase.execute(new SignupCommand(
            "alice@example.com", "short", "alice", true, List.of("01HTERMS01")
        ))).isInstanceOf(BusinessException.class);
    }

    @Test
    void password_hash_called_after_validation() {
        // 검증 통과 시점에 encoder 호출됨 (validation 전 X)
        var order = inOrder(passwordPolicy, encoder);
        when(users.existsByEmail(any())).thenReturn(false);
        when(users.save(any())).thenAnswer(inv -> inv.getArgument(0));

        useCase.execute(validCommand());

        order.verify(passwordPolicy).validate(any(), any(), any());
        order.verify(encoder).encode(any());
    }
}
```

### 4.1 왜 Mockito (진짜 DB 아님)

- 빠름 — DB connection 없이 ms.
- 격리 — UseCase 로직만 검증.
- 의존성 동작 = mock 으로 stub.

### 4.2 왜 `@InjectMocks` 사용

- 생성자 자동 wiring.
- `@RequiredArgsConstructor` (lombok) 와 호환.

### 4.3 왜 `verify(users, never())`

- 행위 검증 — "user 저장 안 됐는지".
- 실패 시나리오에서 side-effect 차단 검증.

---

## 5. 상태 전이 테스트

```java
@ParameterizedTest
@EnumSource(UserStatus.class)
void verifyEmail_only_PENDING(UserStatus status) {
    var u = userWithStatus(status);

    if (status == UserStatus.PENDING_VERIFICATION) {
        u.verifyEmail();   // 성공
        assertThat(u.status()).isEqualTo(UserStatus.ACTIVE);
    } else {
        assertThatThrownBy(u::verifyEmail)
            .isInstanceOf(IllegalStateException.class);
    }
}
```

### 5.1 왜 ParameterizedTest

- 같은 로직을 모든 enum 에.
- 중복 코드 ↓ + 빠진 케이스 발견 ↑.

---

## 6. 도메인 이벤트 테스트

```java
@Test
void domain_events_published_on_state_change() {
    var u = newPendingUser();
    u.pullDomainEvents();   // 초기화

    u.verifyEmail();
    var events = u.pullDomainEvents();

    assertThat(events).hasSize(1).first()
        .isInstanceOfSatisfying(UserVerified.class, e -> {
            assertThat(e.userId()).isEqualTo(u.id());
        });
}
```

---

## 7. 함정 모음

### 함정 1 — Spring context 사용 (`@SpringBootTest`)
도메인 테스트가 수 초 → CI 비대.
→ pure java 단위 테스트.

### 함정 2 — 도메인이 framework 의존
JPA annotation 이 도메인 객체에 → 격리 X.
→ JPA Entity 별도 + 도메인은 pure.

### 함정 3 — 실제 시간 / random 사용
flaky test — 시간 / random 따라 결과 다름.
→ Clock / Random 주입.

### 함정 4 — Mock 과다
모든 의존성 mock → 실제 통합 시 fail.
→ 도메인은 pure, application 만 mock.

### 함정 5 — `when(...).thenReturn(...)` 만 (verify 없음)
호출 안 됐어도 통과.
→ 중요 호출은 verify.

### 함정 6 — assertEquals 사용
가독성 / 메시지 약함.
→ AssertJ.

### 함정 7 — 테스트 이름이 `test1()` / `testSignup()`
무엇 검증인지 불명.
→ `signup_with_duplicate_email_throws_409` 같은 명시.

### 함정 8 — Given-When-Then 구조 없음
가독성 ↓.
→ 명시적 구조 (주석 또는 빈 줄).

### 함정 9 — `assertTrue(x.equals(y))`
실패 메시지 약함.
→ `assertThat(x).isEqualTo(y)`.

### 함정 10 — VO 의 equals / hashCode 안 테스트
HashSet / HashMap 에서 이상 동작.
→ 검증 + record 사용.

### 함정 11 — fixture 가 inline 중복
같은 user 생성 코드가 매 테스트마다.
→ Builder / common helper.

### 함정 12 — Coverage 만 우선 (assertion 없음)
"통과는 했지만 검증은 안 함" — 가짜 coverage.
→ 의미 있는 assertion.

---

## 8. 관련

- [[testing|↑ testing hub]]
- [[integration-tests]]
- [[test-data-fixtures]] — Builder / fixtures
- [[../domain-model/user-aggregate]] · [[../domain-model/value-objects]]
