---
title: "VerificationTokenStatus — 이메일·휴대폰·패스워드리셋 토큰 공통"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:34:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - enum
  - verification-token
---

# VerificationTokenStatus — 이메일·휴대폰·패스워드리셋 토큰 공통

**[[enums|↑ enums hub]]**  ·  관련: [[../email-verification-impl]] / [[../phone-verification-impl]] / [[../password-reset-impl]]

> 이메일 인증 / 휴대폰 인증 / 패스워드 리셋 — 토큰의 생명주기는 **거의 동일**.
> 따라서 **공통 enum 1개** + 각 도메인이 share 하는 패턴. 또는 도메인별 별도 enum.

---

## 1. 값 정의

```java
// src/main/java/com/example/shop/domain/auth/VerificationTokenStatus.java
public enum VerificationTokenStatus {

    /** 발급 직후. 검증 가능. */
    ACTIVE,

    /** 한 번 사용됨 (consume). 재사용 X. */
    USED,

    /** 명시적 무효화 (재발송 / 관리자 / 보안 조치). */
    REVOKED,

    /** 만료 (TTL 지남). cleanup job 이 표시. */
    EXPIRED;

    public boolean isUsable() { return this == ACTIVE; }
    public boolean isFinal()  { return this != ACTIVE; }
}
```

---

## 2. 왜 4개인가

### 2.1 ACTIVE — 시작 상태

당연. 발급 → consume / revoke / expire 중 하나로 이동.

### 2.2 USED vs EXPIRED — 왜 분리

겉으로 보면 둘 다 "더 못 씀". 분리 이유:

| 상태 | 의미 | audit |
| --- | --- | --- |
| USED | 사용자가 정상 사용 완료 | 정상 흐름 |
| EXPIRED | TTL 지남 | 사용자가 시도 안 함 (UX 문제 가능) |

→ **EXPIRED 비율이 높으면 = 메일 도달 실패 / 사용자 어려움**. 분석 가치.

### 2.3 REVOKED — 왜 별도

- **재발송 시 옛 토큰 무효** — USED 아님 (안 썼음)
- **사용자 강제 무효화 시** (관리자 / 보안 사고)
- **`revokeAllActiveForUser(userId)`** 가 일관 처리

```java
// 재발송 시 — 옛 ACTIVE 모두 REVOKED 로
public void revokeAllActiveForUser(UserId userId) {
    tokens.updateStatusByUserAndStatus(userId, ACTIVE, REVOKED);
}
```

→ "이미 USED 됐는데 또 발송됨" 같은 사고 추적 용이.

---

## 3. 단일 enum vs 도메인별 enum

| 방식 | 장점 | 단점 |
| --- | --- | --- |
| **공통 enum** (본 vault 권장) | 중복 X / 일관성 / 패턴 재사용 | 같은 enum 으로 3 종류 의미 |
| 각 도메인 enum | 명확 분리 / 도메인별 의미 추가 가능 | 동일한 값 3번 정의 |

```java
// 옵션 A — 공통 (권장)
public enum VerificationTokenStatus { ACTIVE, USED, REVOKED, EXPIRED }
class EmailVerificationToken {
    private VerificationTokenStatus status;
}
class PhoneVerificationToken {
    private VerificationTokenStatus status;
}

// 옵션 B — 도메인별 (각 도메인 의미 추가 시)
class EmailVerificationToken {
    public enum Status { ACTIVE, USED, REVOKED, EXPIRED }
    private Status status;
}
class PhoneVerificationToken {
    public enum Status { ACTIVE, USED, REVOKED, EXPIRED, MAX_ATTEMPTS_EXCEEDED }
    private Status status;
}
```

→ 휴대폰 인증의 `MAX_ATTEMPTS_EXCEEDED` 같은 도메인 특화 값이 생기면 **B**. 그 외엔 **A**.

**본 vault**: 시작 = **A (공통)**. 차후 분기 필요 시 분리.

---

## 4. 상태 전이

```
            ACTIVE
            /  |  \
           /   |   \
       consume revoke expire (자동 / cleanup)
         ↓     ↓     ↓
        USED REVOKED EXPIRED
```

각 transition 은 도메인 메서드:

```java
// 이메일 인증 토큰
public void consume(Instant now) {
    if (status != ACTIVE) throw new BusinessException(ResponseCode.INVALID_TOKEN);
    if (!now.isBefore(expiresAt)) throw new BusinessException(ResponseCode.EXPIRED_AUTH_CODE);
    status = USED;
}

public void revoke() {
    if (status != ACTIVE) return;     // 이미 final 이면 no-op
    status = REVOKED;
}
```

→ 도메인이 transition 규칙 보장. enum 자체는 transition 모름.

---

## 5. EXPIRED 의 두 가지 처리 방식

### 방식 A — Lazy (조회 시 판단)

DB 의 status 는 ACTIVE 그대로. 조회 시 `expiresAt` 비교로 만료 판단.

```java
public boolean isUsable(Instant now) {
    return status == ACTIVE && now.isBefore(expiresAt);
}
```

장점: 별도 job 없음 / DB 변경 없음
단점: status 가 진실 X (검색 / audit 시 혼동)

### 방식 B — Eager (cleanup job 으로 EXPIRED 마킹)

```java
@Scheduled(cron = "0 */10 * * * *")    // 10분마다
@SchedulerLock(name = "expireVerificationTokens")
public void expireOverdue() {
    int updated = tokens.markExpired(Instant.now());
    log.info("expired {} verification tokens", updated);
}
```

장점: status 가 진실 / audit 명확
단점: job 운영 / DB 부하

**본 vault**: 두 방식 **혼용** — domain method 는 lazy 검증, cleanup job 은 옛 row 의 status 갱신 (감사 / 분석용).

---

## 6. DB / API

### DB

```sql
status VARCHAR(20) NOT NULL CHECK (status IN ('ACTIVE','USED','REVOKED','EXPIRED'))
```

```java
@Enumerated(EnumType.STRING)
@Column(nullable = false, length = 20)
private VerificationTokenStatus status;
```

### API 응답

verification token 자체는 보통 사용자에 노출 X (백엔드 내부). 노출 시:

```json
{ "tokenId": "...", "status": "ACTIVE", "expiresAt": "..." }
```

API 응답 메시지에 "USED" / "EXPIRED" 표시는 ResponseCode 매핑:
- `INVALID_TOKEN` (UNAUTH_002) — USED / REVOKED / invalid
- `EXPIRED_AUTH_CODE` (GONE_001) — EXPIRED 명시

---

## 7. 비교 — RefreshTokenStatus 와 다른 점

| | VerificationTokenStatus | RefreshTokenStatus |
| --- | --- | --- |
| ACTIVE | ✅ | ✅ |
| **USED** (한 번 쓰면 끝) | ✅ | — (refresh 는 ROTATED 로 대체) |
| **ROTATED** (다음으로 넘김) | — | ✅ |
| REVOKED | ✅ | ✅ |
| EXPIRED | ✅ | ✅ |

→ **refresh 는 ROTATED 가 핵심** (도난 감지). verification 은 단일 사용 → USED.

자세히: [[refresh-token-status]].

---

## 8. 함정 모음

### 함정 1 — USED 가 응답에서 ACTIVE 처럼 보임
"인증 완료됐는데 왜 또 안 되지" — UI 가 USED 명확히 표시.

### 함정 2 — REVOKED 의 audit log 없음
관리자 / 시스템이 왜 revoke 했는지 모름. **revoked_reason 컬럼 추가** 권장:

```sql
ALTER TABLE email_verification_tokens
    ADD COLUMN revoked_reason VARCHAR(100),
    ADD COLUMN revoked_at TIMESTAMPTZ;
```

### 함정 3 — Cleanup 없으면 ACTIVE 가 무한 누적
expiresAt 지난 token 이 status=ACTIVE 그대로. DB 폭증. **10분 job 권장**.

### 함정 4 — 같은 user 의 active token 여러 개
재발송 시 옛 토큰 안 무효 → user 가 옛 메일 클릭해도 통과. **재발송 시 revoke 정책**.

### 함정 5 — EXPIRED 와 시간 비교 분리
도메인 method 가 expiresAt 비교, DB status 가 ACTIVE — 일관성 부족. **둘 다 같이 갱신**.

---

## 9. 관련

- [[enums|↑ enums hub]]
- [[refresh-token-status]] — ROTATED 가 추가됨
- [[user-status]] — verifyEmail 의 결과
- [[../email-verification-impl]] / [[../phone-verification-impl]] / [[../password-reset-impl]] — 사용처
