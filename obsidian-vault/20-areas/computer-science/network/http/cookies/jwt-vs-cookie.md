---
title: "JWT vs Cookie — 비교"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T23:00:00+09:00
tags:
  - network
  - http
  - cookies
  - jwt
  - auth
---

# JWT vs Cookie — 비교

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Cookie / JWT / JWT in Cookie 패턴 |

**[[cookies|↑ Cookies]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 두 패턴

### Pattern A — Session Cookie (Stateful)

```
Cookie: sessionid=abc123
↓ 서버가 store 조회
{user_id: 42, expires: ...}
```

### Pattern B — JWT (Stateless)

```
Authorization: Bearer eyJhbGc.eyJzdWIi.signature
↓ 서버가 서명 검증만 — DB 조회 X
{sub: 42, exp: ..., role: "user"}
```

---

## 2. JWT — 무엇

```
header.payload.signature

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9 . eyJzdWIiOiI0MiIsImV4cCI6MTcxNTU4NzIwMH0 . xyz...
```

### 구조
- **Header** — `{"alg":"HS256","typ":"JWT"}` (Base64)
- **Payload** — `{"sub":"42","exp":1715587200,...}` (Base64)
- **Signature** — HMAC-SHA256(header + payload, secret)

### 검증
```python
import jwt
try:
    payload = jwt.decode(token, secret, algorithms=["HS256"])
    user_id = payload["sub"]
except jwt.InvalidTokenError:
    return 401
```

자세히 → [[../auth/auth]]

---

## 3. 전체 비교

| 측면 | Session Cookie | JWT |
| --- | --- | --- |
| **저장 위치 (서버)** | Session Store (Redis / DB) | 없음 — 토큰 자체 |
| **저장 위치 (클라)** | Cookie (HttpOnly) | localStorage / Cookie / Memory |
| **전송** | 자동 (Cookie) | 명시 (Authorization) |
| **상태** | Stateful | Stateless |
| **스케일링** | Store 공유 필요 | Stateless — 쉽다 |
| **무효화** | 즉시 (store 삭제) | 만료까지 (blacklist 필요) |
| **크기** | 작음 (ID 만) | 큼 (claims) — 매 요청 |
| **CSRF** | SameSite + token | Authorization 헤더 — 자연 안전 |
| **XSS** | HttpOnly | localStorage 면 위험 |
| **모바일** | 어려움 | 쉬움 |
| **SSO / Federation** | 어려움 | 표준 (OAuth/OIDC) |
| **Cross-domain** | 어려움 (cookie 한계) | 쉬움 (헤더) |
| **Audit / 모니터링** | 쉬움 (서버 store) | 어려움 (분산) |

---

## 4. 4 가지 패턴

### 4.1 Session Cookie (전통)

```
+로그인 + Set-Cookie: sessionid=abc;
→ 서버 store: {abc: {user_id: 42}}

+요청 + Cookie: sessionid=abc
→ store 조회 → user 42 인증
```

**적합**:
- 전통 웹 (Django/Rails/Spring)
- 단일 도메인
- 즉시 무효화 필요 (보안 중요)

### 4.2 JWT in localStorage (REST API)

```javascript
// 로그인 후
localStorage.setItem('token', jwt);

// 요청 시
fetch('/api/me', {
  headers: {'Authorization': 'Bearer ' + localStorage.getItem('token')}
});
```

**적합**:
- SPA + REST
- Cross-domain
- 모바일 SDK 와 같은 패턴

**위험**:
- **XSS** — JS 가 토큰 훔치기 가능
- 권장 X — 모던에선

### 4.3 JWT in Cookie (HttpOnly)

```http
Set-Cookie: token=<JWT>; HttpOnly; Secure; SameSite=Strict
```

```javascript
// 자동 — fetch credentials: include
fetch('/api/me', {credentials: 'include'});
```

**적합**:
- SPA + 단일 도메인
- XSS 방어 + Stateless

**한계**:
- JWT 가 크기 큼 → 매 요청 부담
- Cross-domain 쿠키 어려움

### 4.4 Hybrid — Session Cookie + Refresh Token

```
Access Token (JWT, 짧음): Memory
Refresh Token: HttpOnly Cookie

Access 만료 → Refresh 로 새 Access
```

**적합**:
- 모든 보안 + UX 균형
- Auth0, Supabase, Firebase 패턴
- 모던 권장

---

## 5. XSS 방어 관점

### Cookie HttpOnly
- JS X — 자동 방어

### localStorage
- JS 가 읽음 — XSS 가 token 훔침
- 어떤 XSS 든 = 즉시 사용자 가장 (impersonation)

### Memory (in-app state)
- 페이지 새로고침 시 사라짐 → Refresh token 필요
- XSS 가 페이지 전체 조작 가능하면 — 어차피 위험

→ **HttpOnly Cookie 가 가장 안전**.

---

## 6. CSRF 방어 관점

### Cookie 자동 첨부
- 브라우저가 자동 → CSRF 가능
- 방어: SameSite=Strict / CSRF token

### Authorization 헤더
- 명시적 — 브라우저 자동 X
- CSRF 자연 안전

### JWT in Cookie
- Cookie 단점 (CSRF) + JWT 장점 결합
- SameSite=Strict 필수

---

## 7. 무효화 (Logout / Revocation)

### Session Cookie
```
DELETE session:abc → 즉시 무효
```

### JWT
- 만료 (exp) 까지 유효 — 즉시 X
- 옵션:
  - **Blacklist** (revoked tokens DB) — stateless 깨짐
  - **짧은 만료 + Refresh** — 만료 후 자연 무효 (5-15 분)
  - **JWT version (jti)** — user 의 `tokens_invalid_before` timestamp
  - **Asymmetric (RS256)** + key rotation — 큰 변경

---

## 8. 크기 영향

### Session Cookie
```
Cookie: sessionid=abc123          (~30 byte)
```

### JWT
```
Authorization: Bearer eyJhbGciOiJ.eyJzdWIiOiI0MiIsImV4cCI6...
                                                         (200-1000+ byte)
```

→ JWT 가 10-30 배 큼. 매 요청 추가 부담.

---

## 9. 사용 사례별 권장

### 단일 도메인 웹 (전통 또는 SPA)
→ **HttpOnly Session Cookie** (Stateful) 또는 **JWT in HttpOnly Cookie**

### Cross-domain SPA + REST
→ **JWT in Authorization header** (단점 인식 — XSS)

### 모바일 + Backend
→ **JWT + Keychain/KeyStore + Refresh Token**

### Microservice / API Gateway
→ **JWT** (Stateless — Gateway 가 검증)

### SSO / Federation
→ **OAuth 2.0 + OIDC + JWT** (표준)

### 즉시 무효화 필요 (금융, 의료)
→ **Session Cookie** 또는 **JWT + Blacklist**

---

## 10. 함정

### 함정 1 — JWT in localStorage
XSS 가 즉시 계정 도난. HttpOnly Cookie 권장.

### 함정 2 — JWT 의 무한 만료
1 시간 + Refresh — 짧게.

### 함정 3 — JWT alg: none
**보안 취약점** — RFC 가 정의했지만 절대 X. 라이브러리가 검증.

### 함정 4 — JWT 의 민감 정보
Payload = Base64 평문 — 누구나 읽음. 비밀번호 / 카드 X.

### 함정 5 — Session Cookie + Cross-domain
복잡 — SameSite, CORS Credentials, Domain. JWT 가 단순.

### 함정 6 — Hybrid 의 복잡함
Access + Refresh + 회전 — 라이브러리 사용.

### 함정 7 — JWT 의 secret 노출
서명 secret 누출 → 모든 토큰 위조 가능. KMS / Vault.

### 함정 8 — Logout 만으로 JWT 무효 X
서버는 만료까지 인정. 클라가 token 삭제만으로 부족 (도난 시).

---

## 11. 학습 자료

- **RFC 7519** (JWT)
- **OWASP JWT Cheat Sheet**
- "Stop using JWT for sessions" — 유명 블로그 (논쟁)
- "JWT vs Cookie" — Auth0 가이드

---

## 12. 관련

- [[cookies]] — Cookies hub
- [[session-cookies]] — Stateful
- [[../auth/auth]] — JWT 깊이
- [[../auth/bearer-token]] — Authorization: Bearer
- [[../security/security]]
