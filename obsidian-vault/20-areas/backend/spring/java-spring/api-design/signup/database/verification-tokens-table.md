---
title: "verification 테이블 3종 — email / phone / password reset"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T03:08:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - database
  - verification
---

# verification 테이블 3종 — email / phone / password reset

**[[database|↑ database hub]]**

> 3 종 토큰 — 공통 패턴 + 각 차이. 같은 "verification" 인데 매체 / 위협 / TTL / 형식이 다른 이유를 명시.

---

## 1. 왜 3 테이블 분리

| 후보 | 왜 안 됨 |
| --- | --- |
| 단일 `verification_tokens` 테이블 + `type` 컬럼 | TTL / brute force 정책 / identifier 가 다 다름. type 별 조건 분기가 SQL/code 곳곳에 → maintainability ↓ |
| Redis 만 사용 (DB 없음) | password_reset 은 분쟁 / audit 필요 (분실 / 도난 시 "누가 reset 했나"). RDB 가 안전 |
| RDB 만 사용 (Redis 없음) | phone code 는 3분 TTL × 분당 발급 폭주 = RDB 인덱스 부담 |

**본 vault 의 선택**
- email_verification_tokens — RDB. URL token, 24h TTL.
- phone_verifications — **Redis 우선** + RDB 옵션. 6-digit, 3분 TTL.
- password_reset_tokens — RDB. URL token, 30분 TTL, 분쟁 audit.

---

## 2. 공통 패턴 (email / password_reset 의 공통점)

```sql
{table_name} (
    id          CHAR(26) PRIMARY KEY,
    user_id     CHAR(26) NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash  CHAR(64) NOT NULL,                  -- SHA-256(raw)
    status      VARCHAR(20) NOT NULL,                -- ACTIVE / USED / REVOKED / EXPIRED
    issued_at   TIMESTAMPTZ NOT NULL,
    expires_at  TIMESTAMPTZ NOT NULL,
    consumed_at TIMESTAMPTZ,                         -- USED 시점

    CONSTRAINT chk_{table}_status
        CHECK (status IN ('ACTIVE', 'USED', 'REVOKED', 'EXPIRED'))
);

CREATE UNIQUE INDEX ux_{table}_token_hash ON {table} (token_hash);
CREATE INDEX ix_{table}_user_status ON {table} (user_id, status);
```

phone 만 다른 컬럼 → §4.

---

## 3. `email_verification_tokens`

```sql
-- V4__create_email_verification_tokens.sql
CREATE TABLE email_verification_tokens (
    id          CHAR(26) PRIMARY KEY,
    user_id     CHAR(26) NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash  CHAR(64) NOT NULL,
    status      VARCHAR(20) NOT NULL,
    issued_at   TIMESTAMPTZ NOT NULL,
    expires_at  TIMESTAMPTZ NOT NULL,                -- 24h
    consumed_at TIMESTAMPTZ,

    CONSTRAINT chk_email_verif_status
        CHECK (status IN ('ACTIVE', 'USED', 'REVOKED', 'EXPIRED'))
);

CREATE UNIQUE INDEX ux_email_verif_tokens_hash ON email_verification_tokens (token_hash);
CREATE INDEX ix_email_verif_tokens_user_status ON email_verification_tokens (user_id, status);
```

### 3.1 컬럼 / 정책의 "왜"

**왜 raw 가 256-bit base64url (URL token)**
- 사용자가 클릭해야 하는 URL → 6-digit 같은 짧은 코드는 brute force 가능. URL token 은 256 bit = brute 불가.
- base64url = URL 안전 (`-`, `_` 사용, `+`, `/`, `=` 안 씀). 메일에 안전하게 임베드.

**왜 TTL 24시간**
- 사용자가 메일 늦게 봐도 (퇴근 후 확인 등) OK 한 여유 시간.
- 너무 짧으면 (1시간) → 재발송 빈도 ↑ → 메일 발송 비용 / SES cost.
- 너무 길면 (7일) → 도난 / 추측 시 영향 시간 ↑.
- 24h = 산업 표준 (Google, Naver, Kakao 모두 비슷).

**왜 단일 사용 (USED 전이)**
- 한 번 클릭 후 다시 들어가면 → 이미 사용된 link 표시 + 거부.
- 메일 forwarding / 우연 클릭 시 인증이 무한 반복되면 audit 부정확.

**왜 user_id 필수 (signup 시점에 user row 이미 존재)**
- 본 vault 의 흐름: signup → status=PENDING_VERIFICATION → email 발송 → 클릭 시 ACTIVE.
- 즉 user row 는 이미 INSERT 됨 → verification 은 status 변경.
- 다른 흐름 (email 인증 후 user 생성) 대비 단순 + race condition ↓.

**왜 ON DELETE CASCADE**
- user hard delete 시 토큰도 같이 정리. 좀비 토큰으로 인증 시도 X.

### 3.2 안 하면 무슨 문제

- token raw 저장 시 → DB 유출 = 모든 메일 link 즉시 사용 가능 (24h 안).
- USED 전이 안 함 → 같은 link 무한 사용 → 사용자가 실수로 두 번 클릭 시 두 번째 클릭이 다른 사용자에 의한 것일 가능성 audit 불가.
- TTL 미설정 → 영구 유효 link → 메일 가로채기 / 옛 메일 archive 노출 시 영구 위험.

---

## 4. `phone_verifications` — 다른 패턴

```sql
-- V5__create_phone_verifications.sql
CREATE TABLE phone_verifications (
    id          CHAR(26) PRIMARY KEY,
    phone       VARCHAR(20) NOT NULL,                 -- 정규화된 번호 (user_id 가 아니라 phone)
    code_hash   CHAR(64) NOT NULL,                    -- SHA-256(6-digit code)
    status      VARCHAR(20) NOT NULL,
    attempts    INTEGER NOT NULL DEFAULT 0,            -- brute force 카운터
    issued_at   TIMESTAMPTZ NOT NULL,
    expires_at  TIMESTAMPTZ NOT NULL,                  -- 3분
    verified_at TIMESTAMPTZ,                            -- VERIFIED 시점

    CONSTRAINT chk_phone_verif_status
        CHECK (status IN ('PENDING', 'VERIFIED', 'EXPIRED', 'REVOKED'))
);

CREATE UNIQUE INDEX ux_phone_verif_code_hash ON phone_verifications (code_hash);
CREATE INDEX ix_phone_verif_phone_status ON phone_verifications (phone, status, issued_at DESC);
```

### 4.1 email 과의 핵심 차이

| 차이 | email | phone | 왜 |
| --- | --- | --- | --- |
| identifier | user_id | phone | 회원가입 전 본인인증 (예: 휴대폰만으로 가입 시작) → user 없는 시점에도 검증 필요 |
| 토큰 형식 | 256-bit URL | 6-digit | SMS 글자 수 제한 + 입력 UX (사용자가 직접 타이핑) |
| TTL | 24h | 3분 | SMS 가 즉시 도착 + brute force 위협 → 짧을수록 안전 |
| attempts 컬럼 | ❌ | ✅ | 6-digit = 100만 경우 → 5회 lock 필수 |
| status enum | ACTIVE/USED/REVOKED/EXPIRED | PENDING/VERIFIED/EXPIRED/REVOKED | 의미 차이 — URL 은 "사용됨" (USED), 코드는 "검증됨" (VERIFIED) |
| 재발송 rate | 1h 3회 | 1h 5회 + 60초 cooldown | SMS 가 더 짧은 cooldown 필요 (사용자가 "안 왔어요" 즉시 재요청) |

### 4.2 왜 6-digit 코드 (URL token 아님)

- **UX**: SMS 받아서 화면에 입력 → 짧을수록 편함. URL 보다 6-digit.
- **SMS 글자 수**: 한국 SMS 90 byte / LMS 2000 byte. URL 첨부 시 단축 URL 필요 + 보안 우려 (피싱).
- **brute force**: 100만 경우의 수 — 단독으론 약함. **attempts lock + 짧은 TTL** 로 보완.

### 4.3 왜 `phone` identifier (user_id 아님)

- 시나리오: 가입 전 본인 인증 → user row 없음 → phone 만 식별자.
- 동일 phone 으로 여러 user 가입 시도 막는 별도 로직 → `users.phone_hash UNIQUE` 와 결합.

### 4.4 왜 `attempts` 카운터

- 6-digit = 100만 경우. 무제한 시도 시 1초당 100회 시도 = 3시간 brute force 가능.
- 5회 실패 시 lock → 사실상 brute force 불가 (5/1,000,000 = 0.0005% 우연 확률 → 운으로도 거의 불가).

**왜 5회 (3 / 10 아님)**
- 3회 = 사용자 오타 (3번 = 너무 빠른 lock, 사용자 불편).
- 10회 = brute force 시도 시간 ↑ (10/100만 = 0.001% 보안 가능 but UX 우선).
- 5회 = 산업 표준 (NCP SENS / NICE 본인인증 등).

### 4.5 왜 Redis 우선 + RDB 옵션

**Redis 가 더 적합한 이유**
- 3분 TTL — RDB 의 cleanup 부담 ↑.
- 분당 폭주 (재발송 / brute force 시도) — Redis 가 throughput ↑.
- 단순 KV — `phone:verify:{phone}` 같은 key 만으로 충분.

**RDB 가 더 적합한 경우**
- 본인 인증의 법적 audit (PASS/NICE 같은 본인 인증) — RDB.
- 분쟁 시 "이 phone 이 언제 어떻게 인증됐는지" 입증 필요.

**Redis 예시**
```
Key:   phone:verify:{phone}
Value: JSON { codeHash, attempts, status, issuedAt }
TTL:   3분
```

```java
// 본 vault: Redis 기본, 영구 audit 필요 시 DB 미러
public void sendCode(String phone, String code) {
    String key = "phone:verify:" + phone;
    var payload = Map.of(
        "codeHash", sha256(code),
        "attempts", "0",
        "issuedAt", Instant.now().toString()
    );
    redis.opsForHash().putAll(key, payload);
    redis.expire(key, Duration.ofMinutes(3));
    sms.send(phone, "[Yule] 인증번호 " + code);
}
```

자세히: [[../phone-verification-impl]].

---

## 5. `password_reset_tokens`

```sql
-- V6__create_password_reset_tokens.sql
CREATE TABLE password_reset_tokens (
    id          CHAR(26) PRIMARY KEY,
    user_id     CHAR(26) NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash  CHAR(64) NOT NULL,
    status      VARCHAR(20) NOT NULL,
    issued_at   TIMESTAMPTZ NOT NULL,
    expires_at  TIMESTAMPTZ NOT NULL,                 -- 30분
    consumed_at TIMESTAMPTZ,

    CONSTRAINT chk_password_reset_status
        CHECK (status IN ('ACTIVE', 'USED', 'REVOKED', 'EXPIRED'))
);

CREATE UNIQUE INDEX ux_password_reset_tokens_hash ON password_reset_tokens (token_hash);
CREATE INDEX ix_password_reset_tokens_user_status ON password_reset_tokens (user_id, status);
```

### 5.1 email 과 거의 동일한 이유

- 둘 다 URL token (256-bit base64url).
- 둘 다 user_id 기반.
- 둘 다 단일 사용 (USED).

### 5.2 왜 TTL 30분 (email 의 24h 아님)

- **위협 모델 차이**:
  - email verification = "이 사람이 이 이메일 주인 맞아?" — 도난 시 가입 우회 정도.
  - password reset = "이 link 클릭 → 비밀번호 변경" — **도난 시 즉시 계정 탈취**.
- 짧은 TTL = 도난 시 사용 가능 시간 ↓.
- 사용자 UX: 비밀번호 재설정은 "지금 당장" 하는 작업 → 30분이면 충분.

### 5.3 왜 별도 테이블 (email 과 합치지 않음)

- 정책이 다름 (TTL / 보안 / monitoring).
- audit / 분쟁 시 "password reset 만" 별도 조회.
- 향후 다른 정책 분기 (예: password reset 은 IP 매칭 강제) 시 schema 자연 확장.

### 5.4 사용 시 도메인 부수 효과 — `revokeAllForUser`

```java
@Transactional
public void confirmPasswordReset(String rawToken, String newPassword) {
    var hash = sha256(rawToken);
    var resetToken = repo.findByTokenHash(hash).orElseThrow(...);
    resetToken.markUsed(now);

    var user = users.findById(resetToken.userId()).orElseThrow();
    user.changePassword(new PasswordHash(encoder.encode(newPassword)));

    // 🚨 매우 중요 — 모든 다른 디바이스 강제 로그아웃
    refreshTokens.revokeAllForUser(user.id(), "PASSWORD_CHANGED");
}
```

**왜 revokeAllForUser**
- 비밀번호 변경 = 보통 도난 의심 / 보안 우려 시 발생.
- 변경 후에도 옛 RT 가 유효하면 → 공격자가 옛 RT 로 계속 access token 갱신 → 의미 없음.
- 모든 디바이스 강제 로그아웃이 보안 best practice.

---

## 6. 3종 비교 표

| 항목 | email | phone | password reset |
| --- | --- | --- | --- |
| TTL | 24h | 3분 | 30분 |
| 저장 | RDB | Redis 우선 (RDB 옵션) | RDB |
| 토큰 형식 | base64url (256bit) | 6-digit 숫자 | base64url (256bit) |
| identifier | user_id | phone (가입 전 가능) | user_id |
| attempts 카운터 | ❌ | ✅ (5회 lock) | ❌ |
| 재발송 rate | 1h 3회 | 1h 5회 + 60s cooldown | 1h 3회 |
| 사용 시 도메인 변경 | `user.status = ACTIVE` | `user.phone = ?, phoneVerifiedAt = now` | `user.passwordHash = ?` + `revokeAllForUser` |
| 위협 모델 | 가입 우회 | brute force code | 계정 탈취 |

자세히: [[../email-verification-model]] · [[../phone-verification-impl]] · [[../password-reset-impl]].

---

## 7. 공통 cleanup job

```sql
-- 매일 새벽 — 만료 7일 지난 모두 삭제
DELETE FROM email_verification_tokens WHERE expires_at < now() - INTERVAL '7 days';
DELETE FROM password_reset_tokens     WHERE expires_at < now() - INTERVAL '7 days';
DELETE FROM phone_verifications        WHERE expires_at < now() - INTERVAL '1 day';
```

**왜 phone 은 1일만**
- TTL 3분 × 분당 발급 폭주 → 일 수만 row 가능.
- audit 가치 낮음 (인증 성공 자체는 user 의 phoneVerifiedAt 에 기록).

**왜 email/password_reset 은 7일**
- 사용자 CS 응대 / 보안 분석 기간.
- 분쟁 시 "이 reset 시도가 언제 있었는지" 입증.

---

## 8. `revokeAllActiveForUser` — 재발송 시

같은 user 의 재발송 시 옛 토큰 무효:

```sql
UPDATE email_verification_tokens
SET status = 'REVOKED'
WHERE user_id = ? AND status = 'ACTIVE';
```

```java
@Modifying(clearAutomatically = true)
@Query("update EmailVerificationTokenJpaEntity t " +
       "set t.status = 'REVOKED' " +
       "where t.userId = :userId and t.status = 'ACTIVE'")
int revokeAllActiveForUser(@Param("userId") String userId);
```

**왜 revoke**
- 재발송 시 옛 link 도 활성 → 사용자가 옛 link 클릭해도 인증 통과 → 가장 최신 link 만 유효해야 함.
- 보안: 옛 link 가 어딘가에 캐싱 / archive 된 경우 영구 노출 위험.

**왜 DELETE 가 아니라 UPDATE → REVOKED**
- audit — "재발송 됐다" 추적 가능.
- cleanup job 이 일괄 삭제하면 됨.

---

## 9. Rate Limit 검사 — `countActiveForUserSince`

```java
@Query("select count(t) from EmailVerificationTokenJpaEntity t " +
       "where t.userId = :userId and t.issuedAt >= :since")
int countActiveForUserSince(@Param("userId") String userId, @Param("since") Instant since);
```

**왜**
- 1시간 내 3회 발급 한도 → 메일 발송 비용 / abuse 방지.
- 검증 안 하면: 누군가가 무한 발송 → SES cost + spam reputation 손상.

**왜 application 단 (Bucket4j 같은 라이브러리 아님)**
- user 별 + 시간 윈도우 = 단순 SQL.
- Bucket4j 는 IP / global rate. user 별 발급 횟수는 DB query 가 단순.

---

## 10. JPA Entity (Email 예시)

```java
@Entity
@Table(name = "email_verification_tokens")
public class EmailVerificationTokenJpaEntity {
    @Id @Column(length = 26) private String id;
    @Column(name = "user_id", nullable = false, length = 26) private String userId;
    @Column(name = "token_hash", nullable = false, length = 64) private String tokenHash;
    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20) private VerificationTokenStatus status;
    @Column(name = "issued_at", nullable = false) private Instant issuedAt;
    @Column(name = "expires_at", nullable = false) private Instant expiresAt;
    @Column(name = "consumed_at") private Instant consumedAt;
}
```

phone_verifications / password_reset_tokens 도 동일 패턴.

---

## 11. 함정 모음 — "이걸 안 하면 X 가 터짐"

### 함정 1 — token raw 저장
DB 유출 = 모든 link / code 즉시 사용 가능. password reset 은 즉시 계정 탈취.
→ SHA-256 hash 만 저장.

### 함정 2 — 단일 사용 (USED) 검증 X
같은 token 여러 번 사용 가능 → audit 부정확 + 메일 forwarding 위험.
→ status 전이 강제 (`status = USED` 후 reject).

### 함정 3 — 재발송 시 옛 토큰 안 무효
ACTIVE 토큰이 user 당 여러 개 → 옛 link 가 영구 활성.
→ revokeAllActiveForUser 강제.

### 함정 4 — phone_verifications 에 user_id 사용
가입 전엔 user 없음 → INSERT 자체 불가 (FK 위반).
→ phone 기반 식별.

### 함정 5 — attempts 카운터 없음 (phone)
6-digit = 100만 경우 → 무제한 시도 시 brute force 가능.
→ 5회 lock + 짧은 TTL.

### 함정 6 — TTL 미설정
영구 유효 link → 옛 메일 archive 노출 시 영구 위험.
→ 모두 명시적 TTL.

### 함정 7 — Cleanup 없음
phone_verifications 일 수만 row → 인덱스 비대 + 검증 느려짐.
→ daily cleanup.

### 함정 8 — UNIQUE token_hash 누락
SHA-256 충돌 (사실상 우주 나이 초과 확률) 또는 application bug 로 중복 생성 시 silent 통과.
→ UNIQUE 인덱스 안전망.

### 함정 9 — password_reset 후 revokeAllForUser 누락
공격자가 옛 RT 로 계속 access token 갱신 → 패스워드 변경 의미 없음.
→ password reset 성공 시 모든 RT revoke 필수.

### 함정 10 — Rate limit 없음
abuse 시 SES cost / SMS cost 폭증 + spam reputation 손상.
→ 1시간 N회 한도 + Bucket4j (IP rate limit) 같이.

### 함정 11 — `phone_hash` 컬럼 (phone 평문 저장)
phone 평문 인덱스 = DB 유출 시 PII. 본 vault: phone 평문 저장 + RDS encryption + 별도 phone_hash (검색용) 정책.

### 함정 12 — 6-digit code 의 `code_hash UNIQUE` 충돌
1000 user 가 동시 인증 시도 시 충돌 확률 (100만 중 1000) — 매우 낮지만 0 아님.
→ UNIQUE 위반 시 application 이 retry (새 code 생성).

---

## 12. 관련

- [[database|↑ database hub]]
- [[../email-verification-impl]] · [[../phone-verification-impl]] · [[../password-reset-impl]]
- [[../enums/verification-token-status]]
- [[../email-verification-model]] — URL token vs 6-digit 비교
- [[email-outbox-table]] — verification 메일 발송 outbox
