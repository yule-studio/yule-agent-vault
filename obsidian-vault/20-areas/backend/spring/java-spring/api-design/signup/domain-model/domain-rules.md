---
title: "Domain Rules — 9 불변식 + 책임 위치"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:19:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - domain-rules
  - invariant
---

# Domain Rules — 9 불변식 + 책임 위치

**[[domain-model|↑ domain-model hub]]**

> 도메인 규칙은 **어디서든 깨지면 안 되는 invariant**. 각 규칙이 어느 layer 의 책임인지 명시.

---

## 1. 규칙 vs 정책

| | 도메인 규칙 (Rule) | 비즈니스 정책 (Policy) |
| --- | --- | --- |
| **변경 빈도** | 거의 안 변함 | 자주 변함 (마케팅 / 운영) |
| **위치** | Aggregate / VO 안 | Application / 설정 |
| **위반 시** | 예외 (`IllegalStateException`) | 비즈니스 응답 (예: 422) |
| **예** | "이메일은 RFC 5322 형식" | "1시간 가입 3회 제한" |

→ **규칙은 도메인 안**, **정책은 Application**.

---

## 2. 9 불변식

### 규칙 1 — Email 은 RFC 5322 형식 + 254자 이내

**위치**: `Email` value object (compact constructor)

```java
public record Email(String value) {
    public Email {
        if (!EMAIL.matcher(value).matches()) throw ...;
        if (value.length() > 254) throw ...;
    }
}
```

**왜 도메인 안**: 어디서든 `new Email(...)` 호출 시 자동 검증. 도메인 안에 invalid Email 존재 X.

### 규칙 2 — Password hash 는 `$argon2` PHC 형식만

**위치**: `PasswordHash` value object

```java
public record PasswordHash(String value) {
    public PasswordHash {
        if (!value.startsWith("$argon2")) throw ...;
    }
}
```

**왜**: 평문이 도메인에 절대 못 들어옴. 타입 안전성.

### 규칙 3 — UserId 는 26자 ULID

**위치**: `UserId` value object

```java
public record UserId(String value) {
    public UserId {
        if (value.length() != 26) throw ...;
        if (!ULID_PATTERN.matcher(value).matches()) throw ...;
    }
}
```

### 규칙 4 — Name 은 1~100자 + blank 금지

**위치**: `User.register(...)` 정적 factory

```java
public static User register(..., String name, ...) {
    if (name == null || name.isBlank() || name.length() > 100)
        throw new IllegalArgumentException("invalid name");
    ...
}
```

**왜 VO 가 아니라 Aggregate?**: name 은 단순 String — 따로 VO 만들기 과함. Aggregate 가 검증.

### 규칙 5 — 신규 가입은 항상 PENDING_VERIFICATION (LOCAL) 또는 ACTIVE (소셜)

**위치**: `User.register(...)` / `User.registerSocial(...)`

```java
public static User register(...) {
    var u = new User(..., UserStatus.PENDING_VERIFICATION, ...);  // 항상 이 상태
    ...
}

public static User registerSocial(...) {
    var u = new User(..., UserStatus.ACTIVE, ..., emailVerifiedAt=now, ...);
    ...
}
```

**왜**: 외부에서 `new User(...)` 못 만듦 (private 생성자). 모든 생성은 factory 거침.

### 규칙 6 — PENDING_VERIFICATION → ACTIVE 만 가능, 다른 경로 X

**위치**: `User.verifyEmail()`

```java
public void verifyEmail(Instant now) {
    if (status == DELETED) throw ...;
    if (emailVerifiedAt != null) return;                      // 멱등
    if (status == PENDING_VERIFICATION) status = ACTIVE;       // 전이
    ...
}
```

**왜**: 상태 머신의 일관성. 어떤 path 로도 ACTIVE 못 됨 — verifyEmail 만 (또는 소셜 register).

### 규칙 7 — DELETED 의 패스워드 변경 X

**위치**: `User.changePassword(...)`

```java
public void changePassword(PasswordHash newHash, Instant now) {
    if (status == DELETED) throw new IllegalStateException("deleted user");
    ...
}
```

**왜**: DELETED 는 종착. 어떤 변경도 의미 X.

### 규칙 8 — 모든 상태 변경은 도메인 이벤트 발행

**위치**: 각 transition method

```java
public void verifyEmail(Instant now) {
    ...
    events.add(new UserEmailVerified(id, email, now));
}
```

**왜**: 외부 부수효과 (이메일 / Kafka / 알림) 가 도메인 흐름과 분리. listener 가 후속 처리.

### 규칙 9 — 소셜 user 는 패스워드 변경 X

**위치**: `User.changePassword(...)`

```java
public void changePassword(PasswordHash newHash, Instant now) {
    if (providerType != SocialProviderType.LOCAL)
        throw new IllegalStateException("social account cannot change password");
    ...
}
```

**왜**: 소셜 user 는 password 없음 (provider 가 관리). 변경 = 의미 X.

---

## 3. 규칙 vs DB Constraint — 이중 방어

| 규칙 | 도메인 (VO/Aggregate) | DB constraint |
| --- | --- | --- |
| email 형식 | ✅ Email VO | — |
| email unique | ✅ existsByEmail 1차 | ✅ `lower(email)` UNIQUE — 진실의 원천 |
| password_hash 형식 | ✅ PasswordHash VO | — |
| name 길이 | ✅ User.register | ✅ `VARCHAR(100)` |
| status enum 값 | ✅ UserStatus | ✅ CHECK constraint |
| 약관 동의 history | ✅ Aggregate | ✅ FK `user_id` |

→ **양쪽 다**. 도메인은 친절한 에러 / 비즈니스 검증, DB 는 마지막 안전망.

---

## 4. 정책 (Application layer)

vs 도메인 규칙:

| 정책 | 위치 |
| --- | --- |
| 가입 1시간 3회 제한 | RateLimiter (Application) |
| 약한 password 차단 (8자 미만 / 유출) | PasswordPolicy (Application) |
| email enumeration 차단 | Controller / Service |
| 가입 후 자동 로그인 | Controller (응답 결정) |
| 신규 user 의 default role | SignupUseCase (Application) |

→ 운영 환경 / 비즈니스 결정. 코드 변경 없이 yaml 로 조정 가능 (이상적).

---

## 5. 규칙의 의도된 누수 vs 누수 사고

### 의도된 — 도메인이 명시적으로

```java
// 도메인에서 외부에 알림
public void delete(Instant now) {
    ...
    events.add(new UserDeleted(id, now));     // 외부 listener 가 처리
}
```

→ event 가 외부 communication 의 채널.

### 사고 — 도메인이 외부 호출

```java
public void verifyEmail() {
    ...
    smsService.send("인증 완료");                 // ❌ 도메인이 SMS 호출
}
```

→ 도메인이 외부 의존. **반드시 이벤트로 분리**.

---

## 6. 규칙 / 정책 결정 가이드

| 질문 | 규칙 | 정책 |
| --- | --- | --- |
| 환경 (dev / prod) 마다 달라야 하나? | ❌ | ✅ — yaml |
| 운영자가 변경 가능해야 하나? | ❌ | ✅ — admin 화면 / config |
| 코드 outside 에 명시 가능? | ❌ | ✅ |
| 어디서든 깨지면 도메인 무결성 깨짐? | ✅ — 도메인 안 | ❌ |
| 비즈니스 결정 / 마케팅 / 운영? | ❌ | ✅ |

예:
- "Password 는 argon2 hash" = **규칙** (어디서든)
- "Argon2 의 m=64MB, t=3, p=4" = **정책** (운영에 따라 강화)

---

## 7. 규칙 위반 시 — 예외 vs Response

```java
// 도메인 규칙 위반 — IllegalStateException / IllegalArgumentException
public void verifyEmail() {
    if (status == DELETED) throw new IllegalStateException("deleted");
}

// Application 단의 비즈니스 검증 — BusinessException
@Transactional
public void signup(...) {
    if (users.existsByEmail(email))
        throw new BusinessException(ResponseCode.DUPLICATE_DATA, "이미 가입");
}
```

→ Controller / `ApiExceptionHandler` 가 둘 다 적절한 HTTP status 로 변환.

- IllegalArgumentException → 400 (잘못된 입력)
- IllegalStateException → 422 (상태 불일치)
- BusinessException → ResponseCode 의 HTTP status

자세히: [[../../common/response-envelope#6 ApiExceptionHandler]].

---

## 8. 규칙 테스트

도메인 unit test 가 모든 규칙 cover:

```java
@Test
void invalid_email_rejected() {
    assertThatThrownBy(() -> new Email("not-an-email"))
        .isInstanceOf(IllegalArgumentException.class);
}

@Test
void deleted_user_cannot_change_password() {
    var user = User.register(...).delete(now);
    assertThatThrownBy(() -> user.changePassword(...))
        .isInstanceOf(IllegalStateException.class);
}

@Test
void social_user_cannot_change_password() {
    var user = User.registerSocial(...);
    assertThatThrownBy(() -> user.changePassword(...))
        .isInstanceOf(IllegalStateException.class)
        .hasMessageContaining("social account");
}
```

→ DB / Spring 없이 ms 단위 빠른 검증.

---

## 9. 함정 모음

### 함정 1 — 규칙을 Service 에 둠
```java
@Service
public class SignupService {
    public void signup(...) {
        if (email.length() > 254) throw ...;   // ❌ — Email VO 가 해야
        ...
    }
}
```
→ 다른 Service 가 같은 규칙 다시 작성. **VO / Aggregate 안에**.

### 함정 2 — DB constraint 만 의존
```sql
email VARCHAR(254)
```
JPA layer 의 검증 없으면 — 257자 입력 시 SQL 실패 (`MysqlDataTruncation`) → 친절한 에러 X. **도메인도 검증**.

### 함정 3 — Bean Validation 만 의존
Controller 단만 검증. 다른 곳 (배치 / 다른 서비스) 에서 직접 호출 시 우회. **도메인 + Bean + DB 3중**.

### 함정 4 — 규칙 변경 시 도메인 + DB 동기화 X
새 규칙 (예: name 최대 50자로 줄임) — 도메인 변경 + DB schema 변경 + 마이그레이션 동시.

### 함정 5 — Aggregate 가 너무 큼 — 규칙도 폭증
User 가 모든 도메인 규칙 책임 — 1000줄. **Aggregate 분리** ([[aggregate-boundaries]]).

### 함정 6 — 정책을 hard-coded
"3회 제한" 이 코드에. 운영자 변경 X. **yaml 또는 admin 설정**.

### 함정 7 — 같은 규칙이 여러 곳
Email 검증이 VO + Service + DTO 셋 다. 단일 출처 (VO) + 다른 곳은 reference.

### 함정 8 — 규칙 변경 추적
"왜 이 규칙?" — git history / ADR (Architecture Decision Record). 도메인 메서드 주석.

---

## 10. 관련

- [[domain-model|↑ domain-model hub]]
- [[user-aggregate]] — 규칙이 위치한 메서드들
- [[value-objects]] — 규칙이 위치한 VO 들
- [[../../common/response-envelope]] — 예외 매핑
- [[aggregate-boundaries]] — Aggregate 가 규칙을 가질 수 있는 범위
