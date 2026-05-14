---
title: "Aggregate Boundaries — 경계 결정 + 왜"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:21:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - aggregate
  - ddd
---

# Aggregate Boundaries — 경계 결정 + 왜

**[[domain-model|↑ domain-model hub]]**

> Aggregate 가 무엇을 들고 무엇을 안 들지의 결정. "왜 약관 동의가 User 안에 없는가" 같은 boundary 의 design rationale.

---

## 1. Aggregate 란 — DDD 정의

> "**Cluster of domain objects** treated as a single unit for the purpose of data changes."
> — Eric Evans, *Domain-Driven Design*

핵심:
- **하나의 root** (Aggregate Root) — 외부에서 보이는 단일 entry point
- **transactional consistency boundary** — Aggregate 안은 트랜잭션 한 번에 일관됨
- **ID 참조로 다른 Aggregate** — 컬렉션으로 가지지 않음 (간접)

---

## 2. auth 도메인의 Aggregate 5종

```
┌──────────────┐  ┌─────────────────┐  ┌────────────────────┐
│ User         │  │ RefreshToken    │  │ EmailVerification  │
│ (Root)       │  │ (Root)          │  │ Token (Root)       │
│              │  │                 │  │                    │
│  - Email     │  │  - userId 참조  │  │  - userId 참조     │
│  - Phone     │  │  - tokenHash    │  │  - tokenHash       │
│  - status    │  │                 │  │                    │
└──────────────┘  └─────────────────┘  └────────────────────┘

┌────────────────────┐  ┌──────────────────────┐
│ PhoneVerification  │  │ PasswordResetToken   │
│ Code (Root)        │  │ (Root)               │
└────────────────────┘  └──────────────────────┘
```

→ **5 개 Aggregate**. 서로 ID 참조 (`userId`) 로만 연결.

---

## 3. 왜 약관 동의가 User Aggregate 안에 없는가

### 3.1 만약 안에 있다면

```java
public final class User {
    private final List<UserTermsConsent> termsConsents;       // ❌
    public void agreeToTerms(Terms terms, Instant now) { ... }
}
```

문제:
1. **User 가 불필요하게 큼** — terms history 가 수십 row → User load 시마다 fetch
2. **약관 1개 추가 = User Aggregate 변경** — 결합도 ↑
3. **다른 도메인 (terms / consent) 의 규칙이 User 에 누수**

### 3.2 분리한 본 vault

```java
// User (Aggregate)
public final class User {
    private final UserId id;
    private final Email email;
    // ... 약관 동의 안 가짐
}

// UserTermsConsent (별도 entity / Aggregate)
public class UserTermsConsent {
    private final UserId userId;          // User 참조 (ID 만)
    private final TermsId termsId;        // Terms 참조 (ID 만)
    private final boolean consented;
    private final Instant agreedAt;
}
```

→ `UserTermsConsentRepository` 가 별도. User 가 load 될 때 fetch X.

**이점**:
- User 가 작고 빠름
- 약관 정책 변경 시 User 영향 X
- terms history 가 폭증해도 User 성능 영향 X

---

## 4. 왜 RefreshToken 이 User Aggregate 안에 없는가

```java
// ❌ 안에 두면
public final class User {
    private final List<RefreshToken> refreshTokens;
    public RefreshToken issueRefreshToken(...) { ... }
}
```

문제:
1. **per-user N개 RefreshToken** — 한 사용자가 디바이스 5개 = 5 row. 자주 변경.
2. **로그인 = User 전체 load** — 비효율
3. **rotation 시 User 도 dirty** — `@Version` 충돌
4. **revokeAllForUser** 가 User 의 메서드여야 하나? 어색.

### 4.1 본 vault 분리

```java
// User 와 RefreshToken 별도 Aggregate
class User { ... }
class RefreshToken {
    private final UserId userId;          // 참조만
    ...
}

interface RefreshTokenRepository {
    void revokeAllForUser(UserId userId);   // RefreshToken 의 책임
}
```

→ RefreshToken 의 변경이 User load 와 무관.

### 4.2 같은 트랜잭션이지만 다른 Aggregate

```java
@Transactional
public LoginResult login(...) {
    var user = users.findByEmail(...);             // User Aggregate
    if (encoder.matches(...)) { ... }
    var rt = rtService.issue(user.id(), ...);       // RefreshToken Aggregate
    return new LoginResult(...);
}
```

→ 한 트랜잭션이 **2 Aggregate** 다룸. **DDD 가 일반적으로 한 트랜잭션 = 한 Aggregate 권장하지만**, 실용적으론 OK (consistency 가 strong 한 경우).

---

## 5. Aggregate 경계의 기준 (결정 가이드)

| 기준 | "안에 둘 것" | "밖으로 뺄 것" |
| --- | --- | --- |
| 같은 트랜잭션에서 항상 변경? | ✅ | ❌ |
| 라이프사이클 같이? | ✅ | ❌ |
| 변경 빈도 비슷? | ✅ | ❌ |
| 외부에서 직접 조회? | ❌ (root 만) | ✅ (자체 root) |
| 컬렉션 크기 무한 증가? | ❌ | ✅ |
| 다른 도메인 영역? | ❌ | ✅ |

### 5.1 User Aggregate 가 들고 있는 것

```java
class User {
    UserId id;
    Email email;
    PhoneNumber phone;        // 1:1, user 의 핵심 정보
    PasswordHash passwordHash; // 1:1
    String name;
    UserStatus status;
    Role role;
    Instant emailVerifiedAt;
    Instant phoneVerifiedAt;
}
```

→ User 의 **고유 1:1 속성** 만. 다른 entity X.

### 5.2 User Aggregate 가 안 가진 것

| Entity | 별도 Aggregate | 이유 |
| --- | --- | --- |
| UserTermsConsent | ✅ | 1:N + 약관 도메인 분리 |
| RefreshToken | ✅ | 1:N + 자주 변경 |
| EmailVerificationToken | ✅ | 1:N + 단명 |
| PhoneVerificationCode | ✅ | 1:N + 단명 |
| PasswordResetToken | ✅ | 1:N + 단명 |
| Order | ✅ | 다른 도메인 |
| LoginHistory | ✅ | 1:N + audit 도메인 |
| UserSession | ✅ | RefreshToken 과 묶일 수도 |

---

## 6. ID 참조 — 컬렉션 X

```java
// ❌ 컬렉션
class User {
    Set<OrderId> orderIds;          // X — 무한 증가
}

// ✅ 다른 Aggregate 가 참조
class Order {
    UserId buyerId;
}

// 조회 시 — Repository
List<Order> orders = orderRepo.findByBuyer(user.id());
```

→ User 가 자기 orders 알 필요 X. **read 시점에 Repository 호출**.

---

## 7. 1 Aggregate = 1 Transaction (이상)

DDD 의 권장:

```
Transaction A: User.verifyEmail
Transaction B: RefreshToken.rotate
```

→ 각 Aggregate 변경은 별도 트랜잭션.

**현실**: signup 시 — User INSERT + UserTermsConsent INSERT 가 같은 트랜잭션 (consistency 강한 경우 OK).

```java
@Transactional
public void signup(...) {
    users.save(user);
    termsConsentRepo.save(consent);    // 2 Aggregate, 1 Tx
}
```

→ DDD purist 는 "ID 만 참조하고 별도 트랜잭션" 권장. 실무: 데이터 일관성 critical 한 경우 같은 Tx OK.

---

## 8. 외부 노출 — Root 만

```java
// 외부 (Controller / Service) 가 User 접근
class UserController {
    User user = users.findById(id);
    user.changeName(...);                       // ✅ Root 메서드
}

// ❌ 직접 내부 collection 접근
user.getOrders();                                // X — Order 는 다른 Aggregate
```

→ Aggregate 안의 entity (UserTermsConsent 같은) 도 외부 직접 노출 X. **Root 가 메서드 통해 노출**.

---

## 9. 경계 변경 시 — 점진적

운영 중 User 가 너무 커서 약관 동의 분리:

```
v1: User { ..., termsConsents: List<...> }
v2: User { ... } + UserTermsConsent (별도) — 같은 트랜잭션
v3: User (별도 service) + Consent (별도 service) — MSA 분리
```

→ 단계적 마이그레이션 가능. 처음부터 적절한 경계 잡기.

---

## 10. 함정 모음

### 함정 1 — User 가 너무 큼
모든 user-related entity 가 User 안에. load 시 N+1 + 메모리 부담. **분리**.

### 함정 2 — Aggregate 간 직접 객체 참조
```java
class Order {
    User buyer;            // ❌ 다른 Aggregate 의 root 를 직접 들고 있음
}
```
→ **ID 참조** 만:
```java
class Order {
    UserId buyerId;        // ✅
}
```

### 함정 3 — Aggregate root 가 internal entity 노출
```java
class User {
    public List<UserTermsConsent> getConsents() { return consents; }   // ❌
}
```
→ 외부에서 직접 조작 가능. **메서드 통해서만**:
```java
public void agreeToTerms(Terms terms, Instant now) { ... }   // 메서드 통해 변경
public boolean hasAgreedTo(Terms terms) { ... }              // query 메서드
```

### 함정 4 — 트랜잭션 1개에 너무 많은 Aggregate
3개 이상 = 검토. 보통 lock 충돌 / 일관성 / 성능 문제.

### 함정 5 — Aggregate 가 다른 Aggregate 의 메서드 호출
```java
class Order {
    public void place(...) {
        var user = userRepo.findById(buyerId);
        user.deduct...                            // ❌ User Aggregate 직접 변경
    }
}
```
→ Application Service 가 조정.

### 함정 6 — 컬렉션 무한
`User.orders` 가 5000 orders → load 시 OOM. 항상 ID 참조 + Repository.

### 함정 7 — Aggregate 가 외부 호출
`User.verifyEmail()` 에서 SMS 발송. 도메인이 외부 의존. **이벤트로**.

### 함정 8 — Aggregate root 가 너무 많은 책임
"User does everything" — 1000줄. **분리** 또는 별도 Aggregate.

---

## 11. 관련

- [[domain-model|↑ domain-model hub]]
- [[user-aggregate]] — User 의 책임 범위
- [[repository-ports]] — 각 Aggregate 의 Repository
- [[domain-events]] — Aggregate 간 communication
- [[../../architecture]] — 계층 / Port-Adapter
