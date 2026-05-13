---
title: "Bearer Token (JWT / OAuth)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T01:45:00+09:00
tags:
  - network
  - http
  - auth
  - bearer
  - jwt
---

# Bearer Token (JWT / OAuth)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | JWT / Access Token / Refresh Token |

**[[auth|↑ Auth]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

`Authorization: Bearer <token>` — "이 토큰을 보유한 자는 누구든 권한 가짐"
(Bearer = 소지자). RFC 6750. OAuth 2.0 표준.

---

## 2. 형식

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI0MiJ9.signature
```

### 보안
- HTTPS 필수
- 토큰 도난 = 사용자 도용
- 짧은 만료 + Refresh Token 권장

---

## 3. JWT — JSON Web Token (RFC 7519)

가장 흔한 Bearer 형식.

```
header.payload.signature
```

### Header (Base64URL)
```json
{"alg": "HS256", "typ": "JWT"}
```

### Payload (Base64URL — 평문)
```json
{
  "sub": "42",            ← Subject (user ID)
  "iat": 1715587200,      ← Issued At
  "exp": 1715590800,      ← Expiration
  "iss": "https://api.example.com",
  "aud": "https://app.example.com",
  "role": "admin"
}
```

### Signature
```
HMAC-SHA256(
    Base64URL(header) + "." + Base64URL(payload),
    secret
)
```

또는 RSA / ECDSA (비대칭).

---

## 4. JWT 의 표준 Claims

| Claim | 의미 |
| --- | --- |
| **iss** (issuer) | 발급자 |
| **sub** (subject) | 주체 (user ID) |
| **aud** (audience) | 청중 (이 토큰의 대상) |
| **exp** (expiration) | 만료 (Unix timestamp) |
| **nbf** (not before) | 유효 시작 |
| **iat** (issued at) | 발급 시간 |
| **jti** (JWT ID) | 토큰 고유 ID |

### 응용 정의 (private claims)
```json
{
  "role": "admin",
  "permissions": ["read", "write"],
  "tenant_id": "abc"
}
```

---

## 5. 알고리즘

### Symmetric — HMAC
- **HS256** — HMAC-SHA-256 (가장 흔함)
- **HS384, HS512**

### Asymmetric — RSA
- **RS256** — RSA + SHA-256
- 서버는 비밀키, 검증은 공개키
- 여러 서비스가 공개키로 검증 가능

### Asymmetric — ECDSA
- **ES256, ES384**
- 더 작음 / 빠름

### EdDSA
- **EdDSA** — Ed25519
- 모던 권장

### `alg: none` — **절대 X**
- RFC 가 정의했지만 보안 취약점
- 라이브러리가 검증

---

## 6. JWT 검증

```python
import jwt
try:
    payload = jwt.decode(
        token,
        SECRET,
        algorithms=["HS256"],
        audience="https://app.example.com",
        issuer="https://api.example.com",
    )
    user_id = payload["sub"]
except jwt.ExpiredSignatureError:
    return 401, "Token expired"
except jwt.InvalidTokenError:
    return 401, "Invalid token"
```

### 검증 항목
1. Signature 일치
2. `exp` 미경과
3. `nbf` 도달
4. `iss`, `aud` 일치
5. `alg` 가 예상

---

## 7. Access Token + Refresh Token 패턴

### 동작

```
1. 로그인 → Access (15 min) + Refresh (7-30 days)
2. API 호출 → Access 사용
3. Access 만료 → Refresh 로 새 Access 받기
4. Refresh 만료 → 재로그인
```

### 응답 (RFC 6749 OAuth 2.0)
```json
{
  "access_token": "eyJhbGc...",
  "token_type": "Bearer",
  "expires_in": 900,
  "refresh_token": "abc...",
  "scope": "read write"
}
```

### Refresh Token Rotation (보안 ↑)
- 매번 Refresh 사용 시 새 Refresh 발급
- 도난 Refresh 가 한 번 사용 후 무효 (theft detection)

---

## 8. JWT 의 저장 위치

자세히 → [[../cookies/jwt-vs-cookie]]

### Option A — HttpOnly Cookie
```http
Set-Cookie: __Host-access=eyJ...; HttpOnly; Secure; SameSite=Strict; Path=/
```
- XSS 안전 ✅
- CSRF 위험 (SameSite + token)
- 단일 도메인 권장

### Option B — Authorization Header
```http
Authorization: Bearer eyJ...
```
- Cross-domain OK
- localStorage 저장 시 XSS 위험
- Memory 저장 (페이지 새로고침 시 사라짐) — Refresh 로 회복

### 권장
- HttpOnly Cookie + SameSite=Strict (단일 도메인)
- 또는 Memory + Refresh Token (SPA)

---

## 9. Opaque Token vs JWT

### Opaque
```
Authorization: Bearer abc123xyz...
```
- 의미 없는 string
- 서버가 DB 조회 → 검증
- 즉시 무효화 가능
- Stateful

### JWT
- Self-contained — DB 조회 X
- Stateless
- 만료까지 무효화 어려움

### 사용
- **Opaque** — 즉시 무효화 필요 (금융 / 의료)
- **JWT** — 스케일링 (Microservice)

---

## 10. Bearer 의 보안

### 도난 시
- 토큰 = 사용자 가장
- 만료까지 사용 가능

### 방어
- **HTTPS 필수**
- **짧은 만료** (15 분 - 1 시간)
- **Refresh Token Rotation**
- **HttpOnly Cookie** (XSS 방어)
- **Audience / Issuer 검증**
- **mTLS** + Token (DPoP 등 고급)

---

## 11. DPoP — Demonstrating Proof of Possession (RFC 9449)

```http
Authorization: DPoP eyJ...
DPoP: eyJ...   ← 키 보유 증명
```

→ 토큰 + 비대칭 키 — 도난해도 키 없이는 사용 X.

OAuth 2.1 / OIDC 모던 권장.

---

## 12. WWW-Authenticate (401 응답)

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer realm="API",
                  error="invalid_token",
                  error_description="The access token expired"
```

### error 값 (RFC 6750)
- `invalid_request`
- `invalid_token`
- `insufficient_scope`

---

## 13. 함정

### 함정 1 — alg: none
라이브러리가 검증. 절대 허용 X.

### 함정 2 — Secret 노출
HS256 의 secret 누출 → 모든 토큰 위조. KMS / Vault.

### 함정 3 — Payload 의 민감 정보
Base64 평문 — 비밀번호 / 신용카드 X.

### 함정 4 — 무한 만료
1 시간 + Refresh 권장.

### 함정 5 — Audience / Issuer 검증 누락
다른 서비스의 토큰 받아서 사용 가능.

### 함정 6 — Logout 의 한계
서버는 만료까지 인정. Blacklist 또는 짧은 만료 + Refresh.

### 함정 7 — Refresh Token 도난
Rotation + Theft Detection.

### 함정 8 — Bearer Token in URL
Query string — 로그 / Referer 노출. 헤더만.

---

## 14. 학습 자료

- **RFC 6750** (Bearer Token)
- **RFC 7519** (JWT)
- **RFC 9449** (DPoP)
- jwt.io — JWT debugger
- OWASP JWT Cheat Sheet

---

## 15. 관련

- [[auth]] — Auth hub
- [[oauth2-flow]] — OAuth 2.0
- [[../cookies/jwt-vs-cookie]]
- [[../../../security-theory/security-theory]]
