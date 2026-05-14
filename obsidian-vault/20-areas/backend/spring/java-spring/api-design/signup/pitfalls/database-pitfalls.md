---
title: "DB / 정합성 함정"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T06:45:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - pitfalls
  - database
---

# DB / 정합성 함정

**[[pitfalls|↑ pitfalls hub]]**

> DB schema / 인덱스 / 마이그레이션의 흔한 실수.

---

## 함정 1 — `email` UNIQUE 만 (lower 없이)

### 무엇

```sql
CREATE UNIQUE INDEX ix_users_email ON users (email);    -- ❌
```

**문제**
- `alice@x.com` 과 `Alice@X.com` 둘 다 통과 = 같은 사용자가 2 계정.
- 사용자가 대소문자 입력 다르게 → 다음 로그인 "계정 없음" 에러.

### 왜 위험

- 사용자 경험 망함 — CS 폭주.
- 같은 사람의 두 계정 → 데이터 분산.
- 분쟁 시 "어느 계정이 진짜?" 모호.

### 해결

```sql
CREATE UNIQUE INDEX ux_users_email ON users (lower(email));

-- 검색
SELECT * FROM users WHERE lower(email) = lower(?);
```

### 왜 application `toLowerCase` 만 부족

- application 코드는 변경됨 — 옛 코드는 변환했지만 새 코드는 안 할 수 있음.
- 다른 클라이언트 (admin script / batch) 가 직접 INSERT 시 우회.
- DB 가 마지막 진실의 원천.

자세히: [[../database/users-table#3.1]].

---

## 함정 2 — Application `existsByEmail` 만 (DB UNIQUE 없음)

### 무엇

```java
public void signup(...) {
    if (users.existsByEmail(email))         // 검증
        throw new EmailAlreadyExistsException();
    users.save(...);                         // INSERT
}
```

**문제 (race condition)**
```
A: existsByEmail → false
B: existsByEmail → false
A: INSERT 성공
B: INSERT — DataIntegrityViolation
```

### 왜 위험

- 두 사용자 동시 가입 시도 — 같은 email 의 두 row.
- 분당 가입 폭주 시 빈번.

### 해결

- DB UNIQUE constraint **필수**.
- `DataIntegrityViolationException` 잡아 409 매핑.

```java
try {
    users.save(user);
} catch (DataIntegrityViolationException e) {
    throw new EmailAlreadyExistsException();    // 409
}
```

---

## 함정 3 — JPA `ddl-auto: update` 운영 사용

### 무엇

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: update           # ❌ 운영에서
```

**문제**
- application 이 자동으로 schema 변경.
- 어떤 변경이 언제 발생했는지 audit X.
- 다중 인스턴스 — 동시 ALTER 충돌.

### 왜 위험

- 예측 불가능한 운영 사고.
- Flyway 와 충돌 — 진실의 원천 깨짐.

### 해결

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate         # entity ↔ schema 일치만 검증
```

- schema 는 Flyway 가 변경.
- application 은 schema 검증만.

---

## 함정 4 — DELETED user 의 email 재가입 막힘

### 무엇

```
1. user "alice@x.com" 가입 → DELETED
2. 다른 사람이 같은 email 로 가입 → 409 (UNIQUE 위반)
```

### 왜 위험

- 옛 user 의 email 영구 사용 불가.
- 메일 재사용 흐름 망가짐 (탈퇴 후 재가입).

### 해결

**옵션 A: anonymize (본 vault default)**
```sql
UPDATE users SET
    email = 'deleted-' || id || '@deleted.example.com',
    status = 'DELETED'
WHERE id = ?;
```

**옵션 B: partial unique index**
```sql
CREATE UNIQUE INDEX ux_users_email_active
    ON users (lower(email))
    WHERE status <> 'DELETED';
```

→ 선택은 정책 따라. anonymize = 단순 + PII 보호.

자세히: [[../database/users-table#6 Soft Delete]].

---

## 함정 5 — `password_hash VARCHAR(60)` (옛 bcrypt 길이)

### 무엇

```sql
password_hash VARCHAR(60)        -- ❌ bcrypt 만 가정
```

**문제**
- argon2id PHC 형식 = ~120 자.
- 저장 시 silent truncate → 로그인 실패.

### 왜 위험

- 가입은 성공 (truncated hash 저장).
- 로그인 시 truncated hash 와 새 hash 매칭 X.
- 사용자가 "분명 가입했는데 로그인 안 됨" CS 폭주.

### 해결

- `VARCHAR(255)` — argon2id + 미래 알고리즘 변경 대비.

---

## 함정 6 — Flyway V 의 SQL 수정

### 무엇

```
V8 적용 후 → "V8 의 컬럼 길이 100 → 200 변경" 시도 → V8 SQL 직접 수정
```

**문제**
- Flyway checksum mismatch.
- staging 정상 (clean DB), prod 에서 배포 fail.

### 왜 위험

- 운영 중단.
- repair 강제 시 → 변경 사항 prod 에 안 반영.

### 해결

- **Forward-only** — 새 V 작성.
- V8 은 그대로, V9 가 ALTER.

자세히: [[../database/migrations#5 Rollback]].

---

## 함정 7 — 큰 ALTER 를 한 마이그레이션에

### 무엇

```sql
-- V8__add_status_and_backfill.sql
ALTER TABLE users ADD COLUMN status VARCHAR(30);
UPDATE users SET status = 'ACTIVE';            -- 1000만 row → 30분 락
```

### 왜 위험

- 30분 락 = 서비스 중단.
- rollback 시 어려움.

### 해결

```sql
-- V8__add_status_column.sql (수 초)
ALTER TABLE users ADD COLUMN status VARCHAR(30);
```

```
별도 application batch (점진적)
  UPDATE users SET status = 'ACTIVE' WHERE status IS NULL LIMIT 1000;
```

자세히: [[../database/migrations#6 Expand-Contract]].

---

## 함정 8 — `@Enumerated(EnumType.ORDINAL)`

### 무엇

```java
@Enumerated(EnumType.ORDINAL)        // ❌
@Column private Role role;
```

```java
public enum Role { USER, SELLER, ADMIN }    // 0, 1, 2
// 나중에 alphabetical sort 정리:
public enum Role { ADMIN, MASTER, SELLER, USER }   // 0, 1, 2, 3
```

### 왜 위험

- DB 의 `0` 이 USER 였는데 ADMIN 으로 해석.
- 모든 user 의 권한이 침묵 변경.

### 해결

```java
@Enumerated(EnumType.STRING)
@Column(length = 20) private Role role;
```

---

## 함정 9 — `created_at` DEFAULT 누락

### 무엇

```sql
created_at TIMESTAMPTZ NOT NULL        -- DEFAULT 없음
```

**문제**
- JPA 가 set 안 하면 NULL → NOT NULL 위반.
- 직접 INSERT (admin script) 시 사고.

### 해결

```sql
created_at TIMESTAMPTZ NOT NULL DEFAULT now()
```

+ JPA `@CreatedDate` 이중 방어.

---

## 함정 10 — `updated_at` 자동 갱신 누락

### 무엇

```java
@Entity
public class User {
    @Column private Instant updatedAt;        // 자동 갱신 없음
}
```

**문제**
- 직접 SQL UPDATE 시 updated_at 안 바뀜.
- "마지막 변경 시각" 추적 X.

### 해결

```java
@LastModifiedDate
@Column(name = "updated_at", nullable = false)
private Instant updatedAt;
```

+ DB trigger 옵션 (가장 robust).

---

## 함정 11 — `phone_hash` 없이 평문 phone 검색

### 무엇

```sql
-- phone 평문 인덱스 + 검색
WHERE phone = '01012345678'
```

**문제**
- DB 백업 유출 = 모든 phone 평문 노출.
- 한국 PIPC 의 "안전성 확보 조치" 위반.

### 해결

- `phone_hash CHAR(64)` 별도 컬럼 (SHA-256).
- 검색은 `WHERE phone_hash = sha256(?)`.
- 평문은 알림톡 발송 직전에만 사용.

자세히: [[../database/encryption-at-rest#4 L2]].

---

## 함정 12 — Soft delete 후 audit 없음

### 무엇

```sql
UPDATE users SET status = 'DELETED' WHERE id = ?;
```

**문제**
- 누가 / 왜 삭제했는지 X.
- CS 분쟁 시 입증 X.

### 해결

- 별도 `user_audit_log` 또는 `deleted_by` / `delete_reason` 컬럼.
- 모든 mutation 의 actor 기록.

자세히: [[../security/audit-logging]].

---

## 함정 13 — `@Version` 누락 (낙관 락)

### 무엇

```java
@Entity
public class User {
    // @Version 없음
}
```

**문제**
- 동시 UPDATE 시 lost update.
- 예: admin 이 status 변경 + 사용자가 name 변경 동시 → 마지막 wins.

### 해결

```java
@Version
@Column(nullable = false) private long version;
```

+ OptimisticLockException retry 정책.

---

## 함정 14 — FK `ON DELETE CASCADE` 없음

### 무엇

```sql
user_id CHAR(26) REFERENCES users(id)         -- CASCADE 없음
```

**문제**
- user hard delete 시 refresh_tokens 좀비 row.
- 코드가 user_id 참조 시 NPE.

### 해결

```sql
user_id CHAR(26) REFERENCES users(id) ON DELETE CASCADE
```

(또는 RESTRICT + 명시 cleanup).

---

## 함정 15 — 인덱스 미사용 (orphan)

### 무엇

```
1. 처음에 추가한 인덱스
2. 코드 리팩토링 — 더 이상 사용 안 함
3. 인덱스 그대로 → INSERT 부담만
```

### 왜 위험

- 인덱스마다 INSERT/UPDATE 추가 비용.
- 인덱스 비대 → 메모리 / 디스크 부담.

### 해결

```sql
-- pg_stat_user_indexes 모니터
SELECT indexrelname, idx_scan
FROM pg_stat_user_indexes
WHERE schemaname = 'public' AND idx_scan = 0;
```

→ idx_scan = 0 인 인덱스 검토 → 안전 검증 후 DROP.

---

## 관련

- [[pitfalls|↑ pitfalls hub]]
- [[../database/users-table]] · [[../database/migrations]] · [[../database/encryption-at-rest]]
- [[security-pitfalls]] — password / 평문 노출
