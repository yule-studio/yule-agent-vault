---
title: "Test Data / Fixtures — Builder + helper"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T06:10:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - testing
  - fixtures
---

# Test Data / Fixtures — Builder + helper

**[[testing|↑ testing hub]]**

> 테스트 데이터의 단일 진실 / 일관성. 매 테스트마다 user 생성 코드 복붙 = maintainability 폭망.

---

## 1. 왜 fixtures 필요한가

### 1.1 안 하면 무슨 문제

```java
// 각 테스트마다 inline 생성 (안 좋은 예)
@Test
void test1() {
    var user = User.register(
        new UserId("01HZX01..."),
        new Email("test1@example.com"),
        new PasswordHash("$argon2id$v=19$m=65536,t=3,p=4$x$y"),
        "test1",
        Instant.parse("2026-05-14T00:00:00Z")
    );
    // ...
}

@Test
void test2() {
    var user = User.register(
        new UserId("01HZX02..."),
        new Email("test2@example.com"),
        new PasswordHash("$argon2id$v=19$m=65536,t=3,p=4$x$y"),
        "test2",
        Instant.parse("2026-05-14T00:00:00Z")
    );
    // ...
}
```

**문제**
- User.register 의 시그니처 변경 = 모든 테스트 수정.
- 중복 코드 → 가독성 ↓.
- 테스트의 핵심 (시나리오) 가 보이지 않음.

### 1.2 Builder 의 해결

```java
@Test
void test1() {
    var user = TestUserBuilder.aPendingUser()
        .withEmail("test1@example.com")
        .build();
    // 시나리오 핵심만 보임
}
```

---

## 2. TestUserBuilder

```java
public class TestUserBuilder {
    private String userId = UlidCreator.getMonotonicUlid().toString();
    private String email = "test@example.com";
    private String passwordHash = "$argon2id$v=19$m=65536,t=3,p=4$abc$xyz";
    private String name = "test user";
    private UserStatus status = UserStatus.PENDING_VERIFICATION;
    private Role role = Role.USER;
    private SocialProviderType providerType = SocialProviderType.LOCAL;
    private Instant createdAt = Instant.parse("2026-05-14T00:00:00Z");

    public static TestUserBuilder aUser() { return new TestUserBuilder(); }

    public static TestUserBuilder aPendingUser() {
        return new TestUserBuilder().withStatus(UserStatus.PENDING_VERIFICATION);
    }

    public static TestUserBuilder anActiveUser() {
        return new TestUserBuilder().withStatus(UserStatus.ACTIVE);
    }

    public static TestUserBuilder anAdminUser() {
        return new TestUserBuilder().withRole(Role.ADMIN).withStatus(UserStatus.ACTIVE);
    }

    public static TestUserBuilder aSocialUser(SocialProviderType provider, String externalId) {
        return new TestUserBuilder()
            .withProviderType(provider)
            .withExternalId(externalId)
            .withPasswordHash(null);   // 소셜 = password 없음
    }

    // setters ...
    public TestUserBuilder withEmail(String e) { this.email = e; return this; }
    public TestUserBuilder withStatus(UserStatus s) { this.status = s; return this; }
    public TestUserBuilder withRole(Role r) { this.role = r; return this; }
    // ...

    public User build() {
        return User.reconstitute(
            new UserId(userId),
            new Email(email),
            new PasswordHash(passwordHash),
            name,
            status,
            role,
            providerType,
            null,                       // externalId
            null,                       // appleSub
            null,                       // emailVerifiedAt
            null,                       // phoneVerifiedAt
            createdAt
        );
    }

    public UserJpaEntity buildEntity() {
        var e = new UserJpaEntity();
        e.setId(userId);
        e.setEmail(email);
        e.setPasswordHash(passwordHash);
        e.setName(name);
        e.setStatus(status);
        e.setRole(role);
        e.setProviderType(providerType);
        e.setCreatedAt(createdAt);
        e.setUpdatedAt(createdAt);
        return e;
    }
}
```

### 2.1 왜 fluent + builder

- 가독성 — `aPendingUser().withEmail("...").build()`.
- 변경 안 한 필드는 default — 시나리오 무관 부분 노출 X.

### 2.2 왜 static factory (`aPendingUser`, `anActiveUser`)

- 자주 쓰는 변형은 named.
- intent 가 코드에 명확.

### 2.3 왜 도메인 객체 + JPA Entity 둘 다 build

- 도메인 테스트 = `build()` 도메인 User.
- DB 테스트 = `buildEntity()` JPA entity.
- Adapter 통해 변환하면 부담 ↑.

---

## 3. 다른 도메인의 Builder

### 3.1 TestTermsBuilder

```java
public class TestTermsBuilder {
    private String code = "service-terms";
    private String version = "v1.0";
    private boolean required = true;
    // ...

    public static TestTermsBuilder serviceTerms() { return new TestTermsBuilder().withCode("service-terms"); }
    public static TestTermsBuilder privacyPolicy() { return new TestTermsBuilder().withCode("privacy-policy"); }
    public static TestTermsBuilder marketing() {
        return new TestTermsBuilder().withCode("marketing").withRequired(false);
    }
}
```

### 3.2 TestRefreshTokenBuilder

```java
public class TestRefreshTokenBuilder {
    private String userId;
    private RefreshTokenStatus status = RefreshTokenStatus.ACTIVE;
    private Instant issuedAt = Instant.now();
    // ...

    public static TestRefreshTokenBuilder anActiveToken(String userId) {
        return new TestRefreshTokenBuilder().withUserId(userId);
    }

    public static TestRefreshTokenBuilder aRotatedToken(String userId, String rotatedToId) {
        return new TestRefreshTokenBuilder()
            .withUserId(userId)
            .withStatus(RefreshTokenStatus.ROTATED)
            .withRotatedToId(rotatedToId);
    }
}
```

---

## 4. Common helpers

```java
public class TestHelpers {

    public static String ulidString() {
        return UlidCreator.getMonotonicUlid().toString();
    }

    public static String validArgon2Hash() {
        return "$argon2id$v=19$m=65536,t=3,p=4$" + base64Random(16) + "$" + base64Random(32);
    }

    public static String randomEmail() {
        return "test+" + ulidString().toLowerCase() + "@example.com";
    }

    public static Clock fixedClock(String isoInstant) {
        return Clock.fixed(Instant.parse(isoInstant), ZoneOffset.UTC);
    }
}
```

### 4.1 왜 helper static

- 모든 테스트에서 즉시 import.
- 자주 사용하는 fixture / utility.

---

## 5. Test profile yml

```yaml
# application-test.yml
spring:
  jpa:
    properties:
      hibernate:
        format_sql: false                  # 테스트 로그 깨끗

logging:
  level:
    org.hibernate.SQL: ERROR              # SQL 로그 끄기 (DEBUG 시만 ON)

auth:
  jwt:
    secret: test-secret-do-not-use-in-prod-12345678901234567890123456789012
    issuer: test
    access-ttl: PT1M                       # 짧게 — 만료 테스트
    refresh-ttl: PT5M
```

### 5.1 왜 별도 yml

- prod / dev 와 격리.
- 테스트 전용 값 (짧은 TTL, dummy secret).

---

## 6. SQL Fixture (`@Sql`)

```sql
-- src/test/resources/test-data/users.sql
INSERT INTO users (id, email, password_hash, name, status, role, provider_type, created_at, updated_at)
VALUES
  ('01HZTESTUSER001', 'alice@example.com', '$argon2id$v=19$m=65536,t=3,p=4$abc$xyz', 'Alice', 'ACTIVE', 'USER', 'LOCAL', now(), now()),
  ('01HZTESTUSER002', 'bob@example.com',   '$argon2id$v=19$m=65536,t=3,p=4$abc$xyz', 'Bob',   'PENDING_VERIFICATION', 'USER', 'LOCAL', now(), now()),
  ('01HZTESTADMIN01', 'admin@example.com', '$argon2id$v=19$m=65536,t=3,p=4$abc$xyz', 'Admin', 'ACTIVE', 'ADMIN', 'LOCAL', now(), now());

INSERT INTO terms (id, code, version, title, content, required, effective_at, use_yn)
VALUES
  ('01HZTERMS01SERVICE',  'service-terms',  'v1.0', '서비스 약관', '...', true,  now(), 'Y'),
  ('01HZTERMS02PRIVACY',  'privacy-policy', 'v1.0', '개인정보 처리방침', '...', true,  now(), 'Y'),
  ('01HZTERMS03MARKETING', 'marketing',     'v1.0', '마케팅 동의', '...', false, now(), 'Y');
```

```java
@Test
@Sql("/test-data/users.sql")
void test_with_fixed_data() {
    // ...
}
```

### 6.1 왜 SQL (Java code 가 아님)

- DB 직접 INSERT — JPA 우회.
- 큰 dataset 시 빠름.

### 6.2 언제 안 적합

- 동적 data (test 별 변형) — Java Builder.

---

## 7. Mocked External API

### 7.1 WireMock — SES / NCP SENS

```java
@SpringBootTest
class EmailVerificationIT {

    @RegisterExtension
    static WireMockExtension wireMock = WireMockExtension.newInstance()
        .options(WireMockConfiguration.options().port(0))
        .build();

    @DynamicPropertySource
    static void overrideSesUrl(DynamicPropertyRegistry registry) {
        registry.add("aws.ses.endpoint", () -> wireMock.baseUrl());
    }

    @BeforeEach
    void stubSes() {
        wireMock.stubFor(post(urlPathEqualTo("/v2/email/outbound-emails"))
            .willReturn(okJson("{\"MessageId\":\"test-msg-id\"}")));
    }

    @Test
    void sends_email() {
        // ...
    }
}
```

### 7.2 왜 WireMock

- 진짜 SES / NCP 호출 X — 비용 + flaky 회피.
- 응답 시나리오 제어 (success / failure / timeout).

### 7.3 대안

- `@MockBean EmailClient` — Spring context 변경. 단 Spring context cache 깨질 가능성.
- Spy + 자체 implementation — 가능하지만 복잡.

---

## 8. 매 테스트 깨끗한 DB

### 8.1 옵션 A: `@Transactional` rollback

```java
@SpringBootTest
@Transactional
class FooIT { ... }
```

**왜 안 됨 (가끔)**
- AFTER_COMMIT listener 안 발동.
- 비동기 / @Async X.

### 8.2 옵션 B: 매 테스트 clean

```java
@AfterEach
void cleanup() {
    userRepo.deleteAll();
    refreshTokenRepo.deleteAll();
    // ...
}
```

**왜**
- 명시적 cleanup.
- AFTER_COMMIT 도 OK.

**문제**
- delete 순서 — FK 의존성.
- truncate vs delete — 성능.

### 8.3 옵션 C: Flyway clean + migrate

```java
@BeforeEach
void clean() {
    flyway.clean();
    flyway.migrate();
}
```

**왜 안 자주 함**
- 시간 수 초.
- 매 테스트마다 = CI 시간 ↑↑.

### 8.4 본 vault 권장

- 단위 테스트 = `@Transactional` rollback.
- Integration with AFTER_COMMIT = 명시 cleanup.

---

## 9. 함정 모음

### 함정 1 — Builder 없이 inline
중복 + 변경 시 모든 테스트 수정.
→ Builder.

### 함정 2 — Builder 가 단일 시점 (변형 X)
모든 변형이 builder method.
→ named factory (`aPendingUser`, `anActiveUser`).

### 함정 3 — 같은 user ID hardcoded (`"01HZ001"`)
다중 user 시나리오 시 충돌.
→ `ulidString()` 매번 random.

### 함정 4 — 진짜 SES / SMS API 호출
비용 + flaky.
→ WireMock / mock.

### 함정 5 — `@Sql` 의 데이터 outdated
schema 변경 시 manual 수정 누락.
→ 가급적 Builder + Java helper.

### 함정 6 — Cleanup 순서 잘못
FK 위반.
→ 의존성 순서 명시.

### 함정 7 — `@Transactional` rollback 의존인데 AFTER_COMMIT 검증
listener 안 발동 → 검증 X.
→ AFTER_COMMIT 검증 시 별도 패턴.

### 함정 8 — Random data 의존 → flaky
"가끔 fail" — random 의존성.
→ fixed seed 또는 deterministic.

### 함정 9 — Time 의존
"오늘 / 내일" — 시스템 시간 따라.
→ Clock 주입 + fixedClock.

### 함정 10 — Test fixture 가 prod schema 와 어긋남
ALTER TABLE 후 fixture 수정 누락.
→ Flyway 가 schema 진실 + fixture 도 Flyway 통한 시드.

### 함정 11 — 너무 큰 fixture
한 테스트가 N 도메인 setup → 가독성 ↓.
→ 필요 최소 fixture.

### 함정 12 — Helper 가 framework 의존
Spring context 시작 시간 ↑.
→ pure java helper.

---

## 10. 관련

- [[testing|↑ testing hub]]
- [[unit-tests]] · [[integration-tests]] · [[test-scenarios]]
- [[../domain-model]] — 도메인 객체 정의
- [[../database]] — JPA entity / Schema
