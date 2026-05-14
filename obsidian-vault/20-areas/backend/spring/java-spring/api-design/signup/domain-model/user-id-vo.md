---
title: "UserId VO — ULID 선택 + 비교"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:11:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - value-object
  - identifier
  - ulid
---

# UserId VO — ULID 선택 + 비교

**[[domain-model|↑ domain-model hub]]**  ·  ← [[value-objects]]

> 모든 도메인 객체의 ID 형식. **ULID** 선택 + 왜 다른 옵션 아닌지.

---

## 1. 코드

```java
// src/main/java/com/example/shop/domain/user/UserId.java
public record UserId(@JsonValue String value) {

    public UserId {
        if (value == null)
            throw new IllegalArgumentException("UserId required");
        if (value.length() != 26)
            throw new IllegalArgumentException("UserId must be ULID (26 chars): " + value);
        if (!ULID_PATTERN.matcher(value).matches())
            throw new IllegalArgumentException("UserId not valid ULID format");
    }

    private static final Pattern ULID_PATTERN =
        Pattern.compile("^[0-9A-HJKMNP-TV-Z]{26}$");          // Crockford Base32
}
```

→ 다른 ID VO (`OrderId`, `ProductId`, `RefreshTokenId`, ...) 도 같은 패턴.

---

## 2. ID 후보 비교

| 후보 | 길이 | 정렬 | 추측 가능 | 분산 | 본 vault |
| --- | --- | --- | --- | --- | --- |
| `Long` (BIGSERIAL) | 8 bytes | ✅ 시간순 | ⚠️ enumeration 가능 (`/users/1, /users/2, ...`) | DB-bound | ❌ |
| `UUIDv4` | 16 bytes (36 char) | ❌ 분산 | ✅ | ✅ | ❌ |
| **ULID** | 16 bytes (26 char) | **✅ 시간순** | **✅** | **✅** | **✅** |
| UUIDv7 | 16 bytes (36 char) | ✅ 시간순 | ✅ | ✅ | ⚠️ |
| NanoID | 가변 (21 char default) | ❌ | ✅ | ✅ | △ |
| Snowflake | 8 bytes (변환 시 19 char) | ✅ 시간순 | ⚠️ | ✅ | △ |

---

## 3. 왜 ULID

### 3.1 ULID 구조

```
01HZJK3M2V ABCDEFGHJKMNPQRS
[timestamp][      random    ]
```

| 부분 | 길이 | 의미 |
| --- | --- | --- |
| timestamp | 10 char (48 bit) | millisecond, 시간순 정렬 보장 |
| random | 16 char (80 bit) | 추측 불가 |

- **timestamp 48 bit** — millisecond, 시간순 정렬 보장
- **random 80 bit** — 추측 불가
- **26 char** (Crockford Base32) — UUID 의 36 char 보다 짧음
- **case-insensitive** — `1` / `I` / `l` / `0` / `O` 혼동 제거 (Crockford 의 의도)

### 3.2 vs Long (BIGSERIAL)

❌ Long 의 문제:
- `/users/1` → `/users/2` enumeration. 가입자 수 노출.
- DB sequence 의존 — sharding 어려움
- 외부 노출 시 비즈니스 정보 추측 (4월 가입 = ID ~1000, 8월 = ~5000)

ULID 는 timestamp 가 있지만 — 시간 정보는 의도된 노출 (created_at 컬럼과 같은 수준). enumeration 은 random 80 bit 가 차단.

### 3.3 vs UUIDv4

❌ UUIDv4 의 문제:
- 완전 random → B-tree 인덱스 **분산** = 쓰기 시 cache miss
- 36 char (`xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`) — 너무 김
- 정렬 X — `ORDER BY id` 가 무의미

ULID 의 timestamp prefix → 시간 순 정렬 → B-tree 인덱스 **순차 쓰기** = 빠름.

### 3.4 vs UUIDv7

UUIDv7 (RFC 9562 / 2024) 도 timestamp + random 구조. ULID 와 거의 동등.

| | ULID | UUIDv7 |
| --- | --- | --- |
| 길이 | 26 char | 36 char |
| 표준 | de facto | RFC 9562 (2024) |
| 라이브러리 성숙도 | 높음 (10+ 년) | 신규 |
| DB 친화 | ✅ | ✅ |

→ 본 vault: **ULID** (짧음 + 성숙). 신규 프로젝트라면 UUIDv7 도 좋은 선택.

### 3.5 vs NanoID

❌ NanoID 의 한계:
- 시간 정렬 X
- 길이 가변 — 명세 일관성 ↓

### 3.6 vs Snowflake

⚠️ Snowflake:
- Twitter 식. 시간순 정렬 OK
- 단 worker_id / datacenter_id 관리 부담 (분산 환경)
- 64-bit Long 이라 짧긴 함

→ 본 vault 규모엔 ULID 가 단순.

---

## 4. ID 생성 — IdGenerator Port

```java
// domain/common/IdGenerator.java
public interface IdGenerator {
    String next();
}
```

```java
// infrastructure/id/UlidIdGenerator.java
@Component
public class UlidIdGenerator implements IdGenerator {
    @Override
    public String next() {
        return ULID.random();        // io.azam.ulidj
    }
}
```

```java
// 사용 (Application)
var user = User.register(
    new UserId(idGenerator.next()),
    new Email(...),
    ...
);
```

> 💡 **왜 port-adapter?**
> 도메인이 외부 라이브러리 (io.azam.ulidj) 의존 X. 추후 UUIDv7 으로 교체 시 Adapter 만 바꿈.

### 4.1 ULID 라이브러리 옵션

| 라이브러리 | 장점 | 비고 |
| --- | --- | --- |
| `io.azam.ulidj` | 단순, 의존성 0 | 본 vault 권장 |
| `de.huxhorn.sulky:sulky-ulid` | 오랜 역사 | 좋음 |
| Spring 의 `UUID` (UUIDv7 — 6+) | 표준 | UUIDv7 사용 시 |

```kotlin
// build.gradle.kts
implementation("io.azam.ulidj:ulidj:5.2.3")
```

---

## 5. DB 매핑

```sql
CREATE TABLE users (
    id  CHAR(26) PRIMARY KEY,            -- ULID 의 고정 26 char
    ...
);
```

```java
@Entity
class UserJpaEntity {
    @Id @Column(length = 26) private String id;
}
```

> 💡 **왜 CHAR(26) vs VARCHAR(26)?**
> PG / MySQL 의 CHAR 는 고정 길이 — 약간의 storage / 인덱스 효율 ↑.
> VARCHAR(26) 도 거의 동등. 본 vault: CHAR (의도 명확).

> 💡 **`@GeneratedValue` X?**
> Hibernate 의 ID 생성기 사용 X — **Application 단에서 발급**.
> 이유: 도메인이 `User.register(id, ...)` 시점에 이미 ID 가져야 됨 (event 발행 / repo 호출 등). DB 가 발급하면 INSERT 후에야 ID 가 생김 = 도메인 흐름 어색.

---

## 6. API 응답

```java
public record SignupResponse(
    String userId,                         // ULID String 그대로
    String email,
    ...
)
```

응답:
```json
{ "userId": "01HZJK3M2VABCDEFGHJKMNPQRS", "email": "..." }
```

→ URL 에도 그대로 (`/api/v1/users/01HZJK3M2VABCDEFGHJKMNPQRS`).

> 💡 **왜 ID 노출 OK?**
> ULID 의 random 80 bit 가 enumeration 차단. 시간 정보는 의도된 노출 (created_at 과 같은 수준).

---

## 7. 정렬 — `ORDER BY id` 의미

```sql
SELECT * FROM users ORDER BY id DESC LIMIT 20;
-- ULID 의 timestamp prefix → 최신 가입 순
```

`created_at` 인덱스 안 만들어도 ID 만으로 시간순 정렬 가능. 인덱스 1개 절약.

단, **같은 millisecond 의 ULID 는 random 순서**. tie-breaker 필요 시 `(id, created_at)` 같이.

---

## 8. UserId 와 다른 ID 패턴

같은 record 패턴으로:

```java
public record OrderId(@JsonValue String value) {
    public OrderId { /* 같은 검증 */ }
}
public record ProductId(@JsonValue String value) { ... }
public record RefreshTokenId(@JsonValue String value) { ... }
public record EmailVerificationTokenId(@JsonValue String value) { ... }
public record PhoneVerificationCodeId(@JsonValue String value) { ... }
public record PasswordResetTokenId(@JsonValue String value) { ... }
```

→ **타입 안전성** — `UserId(orderId.value())` 같은 실수 차단.

```java
// ❌ 컴파일 에러 (다행)
void handle(UserId userId) { ... }
handle(orderId);             // 컴파일 에러
```

> 💡 **왜 모두 다른 type?**
> Primitive `String` 으로 두면 ID 종류 혼동 (어떤 ID 인지 코드 의도 불명).
> 타입을 다르게 = 컴파일러가 검증.

---

## 9. 함정 모음

### 함정 1 — 길이 검증 없음
`new UserId("")` 통과 → DB 에서 fail. **`length() != 26` 거절**.

### 함정 2 — `@GeneratedValue(IDENTITY)` 사용
DB 가 발급 = 도메인 객체 생성 시점에 ID 없음. `User.register` 가 ID 인자 받는 식과 부적합.

### 함정 3 — ULID 의 case
Crockford Base32 = case-insensitive 의도. 그러나 DB 의 `CHAR(26)` 은 case-sensitive 비교 (UPPER vs lower). 일관성:
```java
public UserId {
    value = value.toUpperCase(Locale.ROOT);     // ULID 표준 = 대문자
    ...
}
```
또는 생성 시점에 항상 대문자 (`ULID.random()` 가 보장).

### 함정 4 — Long ID 사용 — enumeration
공격자가 `/api/v1/users/1` 부터 순차 조회 → 사용자 수 노출. ULID 가 차단.

### 함정 5 — UUIDv4 사용 — 인덱스 분산
B-tree 가 random 으로 쓰여 cache miss. PG 의 fragmentation 증가. **시간순 ID** 권장.

### 함정 6 — UserId / OrderId 가 같은 String type
타입 혼동. 별도 record 로 컴파일러가 검증.

### 함정 7 — JSON serialize 시 wrapper
`{ "userId": { "value": "01HZ..." } }` — caller 불편. **`@JsonValue`** 로 String 단순화.

### 함정 8 — 응답 / 로그 에 ID 출처 모름
다른 entity ID 가 섞임 (OrderId 가 UserId 자리에). 타입으로 차단 + 명명 일관.

---

## 10. 관련

- [[domain-model|↑ domain-model hub]]
- [[value-objects]]
- [[../design-decisions]] — ID 정책
- [[../database/users-table]] — `id CHAR(26) PRIMARY KEY`
- [[user-aggregate]] — `User.register(id, ...)` 호출 흐름
