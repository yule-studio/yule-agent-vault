---
title: "ID 전략 — ULID 통일 + 비교"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T03:20:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - database
  - id
  - ulid
---

# ID 전략 — ULID 통일 + 비교

**[[database|↑ database hub]]**

> 모든 테이블의 PK 를 **ULID (CHAR(26))** 으로 통일. UUID / Long / Snowflake 와의 trade-off 명시.

---

## 1. 결정

| 항목 | 선택 |
| --- | --- |
| ID 형식 | **ULID** (Crockford Base32, 26 char) |
| DB 컬럼 | `CHAR(26)` |
| Java 타입 | `String` (VO 로 wrap: `UserId`, `RefreshTokenId`, ...) |
| 적용 범위 | **모든 테이블** (users / refresh_tokens / verification_tokens / email_outbox / terms / user_terms_consent_history) |

---

## 2. ULID 란

```
01HABCDEFG HJKMNPQRSTVWXYZ
[timestamp][   randomness  ]
```

| 부분 | 길이 | 의미 |
| --- | --- | --- |
| timestamp | 10 char (48 bit) | millisecond 단위 unix time |
| randomness | 16 char (80 bit) | 같은 ms 안 monotonic 보장 (라이브러리) |

- **128 bit** (UUID 와 동일)
- **base32 Crockford** — 26 char 표현 (`0-9A-Z`, 일부 제외)
- **앞 10 char = timestamp** — millisecond 단위
- **뒤 16 char = randomness** — 같은 ms 내 monotonic 보장 (라이브러리 지원)

---

## 3. 왜 ULID — 핵심 이유 5

### 3.1 시간 순서 정렬 — DB B-tree 친화

```
id (ULID)                 created_at
01HQXY...A                10:00:00.001
01HQXY...B                10:00:00.002
01HQXY...C                10:00:00.003
```

→ ID 순 정렬 = 시간 순. **인덱스 hot tail** 패턴이 자연. INSERT 시 인덱스 끝에만 쓰기.

UUID v4 와 비교:
```
id (UUID v4 — random)
a3f2c189...        ← 인덱스 곳곳에 INSERT (split / fragmentation)
0b7e5d6f...
ff028a12...
```

→ UUID v4 는 random — B-tree page split 빈발. INSERT throughput ↓ + index size ↑.

### 3.2 URL / 로그 친화

```
ULID:    01HQXY7K3FAB2CDEFGHIJK MNPQR
UUID v4: a3f2c189-4b5e-6f78-9a01-bc23de45f678
```

→ 26 char (UUID 의 36 char 보다 짧음) + 하이픈 없음 (URL 안전).

### 3.3 분산 환경 — 중앙 시퀀스 X

여러 인스턴스 / 워커가 동시에 ID 생성. PostgreSQL 시퀀스 (`BIGSERIAL`) 같은 중앙화 X.

→ MSA 분리, 데이터 import 시 ID 충돌 X.

### 3.4 비밀번호 재설정 / 토큰 — 추측 불가

```java
// password_reset URL
https://app.com/reset?token=01HQXY7K3FAB2CDEFGHIJKMNPQR
```

→ 충분한 randomness (80 bit) — 추측 불가. (timestamp 부분은 노출되어도 무관)

### 3.5 도구 친화

- Java: `Ulid.fast()` (`f4b1c089` 같은 라이브러리)
- PostgreSQL: `CHAR(26)` — 인덱스 / 정렬 OK
- 로깅: 직접 읽음 (UUID 처럼 분리 안 됨)

---

## 4. 비교 — UUID / Long / UUIDv7 / Snowflake

### 4.1 비교표

| 항목 | ULID | UUID v4 | UUID v7 | Long (Auto) | Snowflake |
| --- | --- | --- | --- | --- | --- |
| 크기 | 26 char (16B) | 36 char (16B) | 36 char (16B) | 8B | 8B |
| 시간순 | ✅ | ❌ | ✅ | ✅ | ✅ |
| 분산 생성 | ✅ | ✅ | ✅ | ❌ (중앙 시퀀스) | ✅ (워커 ID 필요) |
| 추측 불가 | ✅ | ✅ | ✅ | ❌ (순차) | ❌ (순차) |
| URL 친화 | ✅ | ❌ (-) | ❌ (-) | ✅ | ✅ |
| 표준 | 비공식 | RFC 4122 | RFC 9562 | DB | Twitter |
| 도구 지원 | 중 | ✅✅ | 신규 | ✅✅ | 중 |

### 4.2 왜 ULID > UUID v4

- 시간순 + B-tree 친화
- 26 char (URL 짧음)
- 도구 지원 충분

### 4.3 왜 ULID > UUID v7

- UUID v7 도 시간순 (RFC 9562, 2024) — 좋음
- 단 ULID 가 26 char 로 더 짧음
- ULID 의 도구 / 인지도 충분 (2016 이후)
- v7 마이그레이션 시 — 도구 지원 좀 더 보면 v7 도 OK

> 💡 **신규 프로젝트라면 UUID v7 도 매우 좋은 옵션** — 표준 + 시간순. ULID 와 거의 동등.

### 4.4 왜 ULID > Long (Auto)

- **노출 시 추측 가능** — `id=12345` 다음은 `id=12346` (보안 / 정보 누출)
- 분산 생성 X — 마이그레이션 / MSA 분리 어려움
- DB 시퀀스 의존 — 백업 / 복구 복잡

### 4.5 왜 ULID > Snowflake

- Snowflake 는 **워커 ID 관리** 필요 (Zookeeper / 환경변수)
- 인프라 부담
- 단 — Snowflake 는 8 byte (ULID 의 16 byte 의 절반) → 매우 큰 테이블에선 유리

### 4.6 ULID 의 단점

| 단점 | 완화 |
| --- | --- |
| 비표준 (UUID 처럼 RFC 없음) | 충분히 안정 (8년+) |
| 26 char = 8 byte (Long) 의 3배 | 본 vault 규모 무관 |
| timestamp 노출 (생성 시점 추정 가능) | password reset token 등은 무관, 일반 객체도 created_at 이미 있음 |
| monotonic 보장은 라이브러리 의존 | `Ulid.fast()` 등 monotonic 지원 라이브러리 사용 |

---

## 5. Java 라이브러리

### 5.1 추천 — `f4b1c089/ulid-creator`

```xml
<dependency>
    <groupId>com.github.f4b6a3</groupId>
    <artifactId>ulid-creator</artifactId>
    <version>5.2.3</version>
</dependency>
```

```java
import com.github.f4b6a3.ulid.Ulid;
import com.github.f4b6a3.ulid.UlidCreator;

// 생성
String id = UlidCreator.getUlid().toString();    // "01HQXY7K3FAB2CDEFGHIJKMNPQR"

// monotonic — 같은 ms 안에서 순서 보장
String id2 = UlidCreator.getMonotonicUlid().toString();

// 파싱
Ulid ulid = Ulid.from("01HQXY...");
Instant ts = ulid.getInstant();
```

### 5.2 VO wrap

```java
public record UserId(String value) {
    public UserId {
        if (value == null || value.length() != 26)
            throw new IllegalArgumentException("invalid ULID: " + value);
    }
    public static UserId generate() {
        return new UserId(UlidCreator.getMonotonicUlid().toString());
    }
    public Instant createdAt() {
        return Ulid.from(value).getInstant();
    }
}
```

자세히: [[../domain-model/user-id-vo]].

### 5.3 ID 별 VO

| 도메인 | VO |
| --- | --- |
| User | `UserId` |
| RefreshToken | `RefreshTokenId` |
| VerificationToken (email/phone/password) | `VerificationTokenId` |
| EmailOutbox | `EmailOutboxId` |
| Terms | `TermsId` |
| UserTermsConsentHistory | (audit, 자체 PK 만 — VO 안 만들어도 됨) |

→ **각 Aggregate ID 는 자체 VO**. 타입 안전성.

---

## 6. DB schema 예시

```sql
-- users
CREATE TABLE users (
    id CHAR(26) PRIMARY KEY,
    ...
);

-- refresh_tokens (FK 도 CHAR(26))
CREATE TABLE refresh_tokens (
    id CHAR(26) PRIMARY KEY,
    user_id CHAR(26) NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

→ FK 도 동일 `CHAR(26)`. 일관성.

---

## 7. JPA 매핑

```java
@Entity
@Table(name = "users")
public class UserJpaEntity {
    @Id
    @Column(length = 26)
    private String id;
    ...
}

@Entity
@Table(name = "refresh_tokens")
public class RefreshTokenJpaEntity {
    @Id @Column(length = 26) private String id;
    @Column(name = "user_id", length = 26) private String userId;     // FK 도 String
    ...
}
```

> 💡 **왜 `@GeneratedValue` 안 씀?**
> ULID 는 **도메인 (Application service / Aggregate) 가 생성** — DB 가 아님.
> `UserId.generate()` 가 ID 부여. JPA 는 그대로 INSERT.

---

## 8. 외부 노출 — REST API

```http
GET /v1/users/01HQXY7K3FAB2CDEFGHIJKMNPQR

→ 26 char (UUID 의 36 보다 짧음, 하이픈 없음)
→ URL 친화
```

> 💡 **내부 ID 노출은 OK** — ULID 의 randomness 가 추측 불가. (Long ID 와 다름)

---

## 9. ID 변경 / 마이그레이션 정책

### 9.1 처음부터 ULID 로

신규 프로젝트 = 무조건 ULID (or UUID v7). 운영 중 변경 = 매우 비용.

### 9.2 운영 중 Long → ULID

```sql
ALTER TABLE users ADD COLUMN id_new CHAR(26);
UPDATE users SET id_new = generate_ulid();       -- 또는 application 단
-- 모든 FK 변경
-- old PK drop, new PK
```

→ **Big bang 또는 점진적 — downtime / dual-write 필요**. 신중.

---

## 10. 함정 모음

### 함정 1 — `VARCHAR(36)` 으로 UUID 했는데 ULID 도 같이
`CHAR(26)` 으로 통일. 길이 다르면 join / 인덱스 비효율.

### 함정 2 — DB AUTO_INCREMENT 인데 분산 시스템
중앙 DB 의존. MSA 분리 시 ID 충돌.

### 함정 3 — Snowflake 인데 워커 ID 관리 X
워커 환경변수 누락 시 ID 충돌. 인프라 부담.

### 함정 4 — UUID v4 + B-tree fragmentation
대규모 INSERT 환경에서 인덱스 split + IO ↑.

### 함정 5 — ULID monotonic 미보장
`UlidCreator.getUlid()` 만 사용 시 — 같은 ms 안에 순서 random. **`getMonotonicUlid()`** 사용.

### 함정 6 — 응답 DTO 가 Long ID
순차 노출 = 추측 가능. ULID 로 변환 또는 hash.

### 함정 7 — ULID vs UUID 혼용
일관성 X. 한 프로젝트 = 한 형식.

### 함정 8 — `String` 으로만 받음 — 타입 안전성 X
```java
void process(String userId, String orderId) { ... }    // 헷갈림
process(orderId, userId);                              // 컴파일 통과 (😱)
```
→ VO (`UserId`, `OrderId`) 로 명시.

---

## 11. 관련

- [[database|↑ database hub]]
- [[../domain-model/user-id-vo]] — UserId VO
- [[../design-decisions]] — Why ULID (전체 ADR)
- [[users-table]] · [[refresh-tokens-table]] · [[verification-tokens-table]] · [[email-outbox-table]]
