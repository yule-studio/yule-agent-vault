---
title: "RefreshTokenStatus — refresh 토큰 lifecycle"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:36:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - enum
  - refresh-token
---

# RefreshTokenStatus — refresh 토큰 lifecycle

**[[enums|↑ enums hub]]**  ·  관련: [[../login-impl]] / [[../token-refresh-impl]]

> verification token 과 비슷하지만 **ROTATED** 가 추가. rotation + reuse detection 핵심.

---

## 1. 값 정의

```java
// src/main/java/com/example/shop/domain/auth/RefreshTokenStatus.java
public enum RefreshTokenStatus {

    /** 발급 직후. 갱신 / logout 가능. */
    ACTIVE,

    /** rotation 으로 새 토큰 발급됨. 이 토큰은 더 사용 X. */
    ROTATED,

    /** 명시적 무효화 (logout / 패스워드 변경 / 보안 사고). */
    REVOKED,

    /** TTL 지남 (보통 14일). */
    EXPIRED;

    public boolean isUsable() { return this == ACTIVE; }
    public boolean isFinal()  { return this != ACTIVE; }
}
```

---

## 2. ROTATED 가 핵심 — 왜 필요한가

### 2.1 일반 토큰 (USED) 과의 차이

```
[verification token]
  ACTIVE → consume() → USED              (한 번 쓰고 끝)

[refresh token]
  ACTIVE → rotate() → ROTATED            (한 번 쓰고 새 RT 발급)
              ↓
            new RT (ACTIVE)
```

→ verification 은 "끝", refresh 는 "이어짐".

### 2.2 Reuse Detection — ROTATED 의 진짜 목적

```
공격자가 RT_v1 탈취
       ↓
정상 user 가 RT_v1 사용 → ACTIVE → ROTATED, 새 RT_v2 발급
       ↓
공격자가 탈취한 RT_v1 시도
       ↓
서버: 발견 — RT_v1.status = ROTATED  ← "reuse 감지!"
       ↓
🚨 모든 user 의 RT REVOKE (도난 의심)
```

→ 만약 RT_v1.status 가 그냥 USED 였다면 정상 사용인지 도난인지 구분 불가. **ROTATED** 상태가 "이미 다음으로 넘어갔다" 표시.

자세히: [[../token-refresh-impl#3 Reuse Detection]].

---

## 3. 왜 ROTATED 와 USED 를 같은 이름 (예: USED) 으로 안 하는가

옵션:
```java
public enum Status { ACTIVE, USED, REVOKED, EXPIRED }
```

이렇게 통합하면 — verification / refresh 둘 다 사용 가능. 단점:

| 문제 | 설명 |
| --- | --- |
| reuse detection 의 의미 부족 | "USED 됐다" 만으로는 정상 + 도난 구분 X (다음 토큰 ID 추적 필요) |
| 도메인 의도 불명 | "이미 사용됨" vs "rotation 됐음" — 다른 의미 |

→ **별도 enum** 또는 같은 enum 에 ROTATED 값 추가. 본 vault: 별도 enum.

대신, **`rotatedToId` 컬럼 도 같이** 보관:

```sql
CREATE TABLE refresh_tokens (
    ...
    status         VARCHAR(20) NOT NULL,
    rotated_to_id  CHAR(26),           -- ROTATED 일 때 다음 RT 의 ID
    ...
);
```

→ ROTATED 된 RT 가 어떤 RT 로 이어졌는지 chain 추적 가능 (감사 / debugging).

---

## 4. 상태 전이

```
                ACTIVE
                /  |  \
               /   |   \
          rotate revoke expire
            ↓      ↓      ↓
        ROTATED REVOKED EXPIRED
```

도메인 메서드:

```java
public void rotate(RefreshTokenId newId, Instant now) {
    if (status != ACTIVE) throw new IllegalStateException("not active: " + status);
    if (!now.isBefore(expiresAt)) throw new IllegalStateException("expired");
    this.status = ROTATED;
    this.rotatedToId = newId;
}

public void revoke() {
    if (status != ACTIVE) return;    // 이미 final 이면 no-op (멱등 logout)
    status = REVOKED;
}
```

---

## 5. ROTATED row 의 보존 정책

| 정책 | 설명 |
| --- | --- |
| **즉시 삭제** | rotation 즉시 DELETE | reuse detection 무력화 — ❌ |
| **TTL 후 삭제** (기본) | expiresAt + 7일 후 cleanup | 본 vault — 추적 + 정리 균형 |
| **영구 보존** | audit 강한 도메인 | 금융 / 의료 |

```sql
-- cleanup job
DELETE FROM refresh_tokens
WHERE expires_at < now() - INTERVAL '7 days'
  AND status IN ('ROTATED', 'REVOKED', 'EXPIRED');
```

→ ROTATED row 도 충분 시간 보존 후 cleanup.

---

## 6. 다중 디바이스 — 같은 user 의 RT 여러 개

```sql
SELECT * FROM refresh_tokens WHERE user_id = ? AND status = 'ACTIVE';
-- 디바이스 별로 ACTIVE 1개씩 = N 개 (정상)
```

각 디바이스가 자기 RT 만 rotation. 한 디바이스의 reuse detection 발생 시:

| 정책 | 영향 |
| --- | --- |
| 해당 RT chain 만 REVOKE | 다른 디바이스 영향 X (UX 우선) |
| **user 의 모든 RT REVOKE** (본 vault) | 모든 디바이스 강제 로그아웃 (안전 우선) |

본 vault — **모든 RT REVOKE**. 도난 의심 시 안전 우선.

---

## 7. DB

```sql
CREATE TABLE refresh_tokens (
    id                 CHAR(26) PRIMARY KEY,             -- jti
    user_id            CHAR(26) NOT NULL REFERENCES users(id),
    token_hash         CHAR(64) NOT NULL,                 -- SHA-256
    device_fingerprint VARCHAR(255),
    status             VARCHAR(20) NOT NULL,
    rotated_to_id      CHAR(26),
    issued_at          TIMESTAMPTZ NOT NULL,
    expires_at         TIMESTAMPTZ NOT NULL
);
CREATE UNIQUE INDEX ux_refresh_tokens_hash ON refresh_tokens (token_hash);
CREATE INDEX ix_refresh_tokens_user_status ON refresh_tokens (user_id, status);
CREATE INDEX ix_refresh_tokens_expires ON refresh_tokens (expires_at);
```

```java
@Enumerated(EnumType.STRING)
@Column(nullable = false, length = 20)
private RefreshTokenStatus status;
```

---

## 8. API 응답에서 노출 정책

refresh token 자체는 사용자에 raw 형태로만 노출. status / metadata 는 **외부 노출 X**.

내부 admin 화면 (사용자의 active session 목록 — "다른 기기에서 로그아웃" 기능):

```http
GET /api/v1/me/sessions

200 OK
{
  "result": [
    { "id": "01HZ...", "device": "iPhone 14", "ip": "112.x.x.x", "lastActiveAt": "..." },
    { "id": "01HZ...", "device": "Chrome (Mac)", "ip": "210.x.x.x", "lastActiveAt": "..." }
  ]
}
```

→ status 자체는 응답에 안 보내고 `ACTIVE` 인 것만 list.

---

## 9. 함정 모음

### 함정 1 — ROTATED 즉시 삭제
reuse detection 무력화. **TTL 후 삭제**.

### 함정 2 — `rotatedToId` 컬럼 없이 ROTATED 만
chain 추적 불가. **rotated_to_id 같이**.

### 함정 3 — refresh 검증 시 token_type claim 무시
access token 으로 refresh endpoint 호출 통과. **`isRefresh(token)` 명시 검증**.

### 함정 4 — Reuse detection 시 user 의 모든 RT 안 죽임
"이 chain 만" 죽이면 같은 user 의 다른 디바이스가 영향 X = 도난 detection 의미 ↓. **모든 RT REVOKE**.

### 함정 5 — refresh 가 사용자 ID claim 가짐
JWT 디코드만 하면 user ID 노출. **refresh 는 opaque (random) + 서버 매핑**.

### 함정 6 — Logout 시 status 만 REVOKED, 토큰 자체는 클라에
클라가 또 사용 시도 — 서버: ACTIVE 아님 → 401. OK 하지만 사용자 혼동.

---

## 10. 관련

- [[enums|↑ enums hub]]
- [[verification-token-status]] — USED 와 ROTATED 차이
- [[../login-impl]] — 발급 / rotate
- [[../token-refresh-impl]] — reuse detection
