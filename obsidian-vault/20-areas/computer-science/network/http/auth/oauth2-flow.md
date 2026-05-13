---
title: "OAuth 2.0 Flow"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T01:50:00+09:00
tags:
  - network
  - http
  - auth
  - oauth
  - oidc
---

# OAuth 2.0 Flow

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 4 grant + PKCE + OIDC |

**[[auth|↑ Auth]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

**위임 인증** — 사용자가 자기 비밀번호 안 주고 다른 응용에 자기 자원 접근 허락. RFC 6749 (2012).

---

## 2. 4 명의 역할

| 역할 | 의미 | 예 |
| --- | --- | --- |
| **Resource Owner** | 자원 소유자 | 사용자 |
| **Client** | 요청자 | 제3 사 응용 |
| **Authorization Server** | 인증 / 토큰 발급 | Google / Auth0 |
| **Resource Server** | 자원 보유 | Google Drive API |

---

## 3. Grant Types (4 + 추가)

### 3.1 Authorization Code (가장 흔함)
- 웹 / 모바일 SPA
- **PKCE** 와 함께 (모던 필수)

### 3.2 Client Credentials
- Service-to-Service
- 사용자 X — 응용 자체 인증

### 3.3 Refresh Token
- Access 갱신
- 별도 grant 가 아닌 부수 흐름

### ~~Implicit~~ (deprecated)
- SPA 옛 — PKCE 로 대체

### ~~Resource Owner Password~~ (deprecated)
- 비밀번호 직접 — 위임의 의미 없음

### 3.4 Device Code (RFC 8628)
- TV / 콘솔 / IoT — 키보드 X 디바이스

---

## 4. Authorization Code Flow + PKCE

### 단계

```
1. Client → User: Authorize URL 로 redirect
   GET https://auth.example.com/authorize?
       response_type=code&
       client_id=abc&
       redirect_uri=https://app.com/callback&
       scope=read+write&
       state=xyz&
       code_challenge=...&
       code_challenge_method=S256

2. User: 로그인 + 동의

3. Auth Server → Client (redirect):
   GET https://app.com/callback?code=AUTH_CODE&state=xyz

4. Client → Auth Server: Token 요청
   POST /token
   grant_type=authorization_code
   code=AUTH_CODE
   redirect_uri=https://app.com/callback
   client_id=abc
   code_verifier=...        ← PKCE

5. Auth Server → Client:
   {
     "access_token": "...",
     "refresh_token": "...",
     "expires_in": 3600,
     "token_type": "Bearer"
   }

6. Client → Resource Server:
   Authorization: Bearer ...
```

---

## 5. PKCE (RFC 7636)

### 동기
- Authorization code 도난 시 (mobile / SPA) 토큰 받기 가능
- Public client (clientSecret 보관 X) 보호

### 동작

```
1. Client: code_verifier 생성 (random 43-128 byte)
2. Client: code_challenge = SHA256(code_verifier) → Base64URL
3. /authorize 에 code_challenge + S256
4. /token 에 code_verifier 보냄
5. Auth Server: SHA256(verifier) == challenge ?
```

→ Code 도난해도 verifier 없으면 토큰 X.

### 모던 — 모든 OAuth 에 PKCE 권장
- OAuth 2.1 — PKCE 강제

---

## 6. Scope

```
scope=read write profile email
```

### 의미
- 허락 범위 — 응용이 요청, 사용자가 동의
- "이 응용은 당신의 이메일 read 가능"

### 표준 scope (OpenID Connect)
- `openid` — OIDC 활성
- `profile` — 사용자 프로필
- `email` — 이메일
- `offline_access` — Refresh Token

---

## 7. State 파라미터

```
authorize URL: state=xyz...
callback URL: state=xyz...
```

### 동기
- CSRF 방어
- 클라가 random state 생성 → 검증

---

## 8. OpenID Connect (OIDC, RFC 9068)

OAuth 2.0 위에 **인증 (identity)** 추가.

```
OAuth 2.0  — 인가 (authorization)
OIDC       — 인증 (authentication) + OAuth
```

### 차이
- OAuth 응답: Access Token 만
- OIDC 응답: Access Token + **ID Token (JWT)**

### ID Token
```json
{
  "iss": "https://auth.example.com",
  "sub": "user-42",
  "aud": "client-abc",
  "exp": ...,
  "iat": ...,
  "email": "alice@example.com",
  "name": "Alice"
}
```

### Userinfo Endpoint
```http
GET /userinfo
Authorization: Bearer <access_token>

→ {"sub": "user-42", "email": "...", "name": "..."}
```

---

## 9. Client Credentials Flow

```
POST /token
grant_type=client_credentials
client_id=...
client_secret=...
scope=read

→ {access_token, expires_in, ...}
```

### 사용
- Microservice → Microservice
- API Gateway 의 외부 API 호출
- Cron job

### 사용자 X
- "사용자 위임" 의 OAuth 가 아닌 단순 API Key 대체

---

## 10. Device Code Flow (RFC 8628)

```
1. Device → Auth Server:
   POST /device_authorization
   client_id=...

2. Auth Server: device_code, user_code, verification_uri

3. Device: 사용자에게 표시
   "Go to https://example.com/device and enter ABC-123"

4. Device → Auth Server: 폴링 (5 초 마다)
   POST /token
   grant_type=urn:ietf:params:oauth:grant-type:device_code
   device_code=...
   client_id=...

5. 사용자가 다른 디바이스에서 로그인 + 동의

6. Auth Server: access_token 응답
```

### 사용
- Apple TV / Roku 등 TV
- CLI (gh auth login)

---

## 11. Refresh Token Flow

```
POST /token
grant_type=refresh_token
refresh_token=...
client_id=...

→ {access_token, refresh_token (rotation), expires_in}
```

### Refresh Token Rotation
- 매번 새 Refresh 발급
- 옛 Refresh 무효 — 도난 감지
- Auth0 / Okta 표준

---

## 12. Discovery — Well-Known URLs

```
https://auth.example.com/.well-known/openid-configuration
```

```json
{
  "issuer": "https://auth.example.com",
  "authorization_endpoint": "https://auth.example.com/authorize",
  "token_endpoint": "https://auth.example.com/token",
  "userinfo_endpoint": "https://auth.example.com/userinfo",
  "jwks_uri": "https://auth.example.com/.well-known/jwks.json",
  "scopes_supported": [...],
  ...
}
```

→ 클라가 endpoint 자동 발견 + 검증.

---

## 13. OAuth 2.1 (RFC draft, 2024+)

### 변경
- PKCE 강제
- Implicit / Password Grant 제거
- Refresh Token Rotation 권장
- HTTPS 강제
- 더 안전한 default

---

## 14. 흔한 Auth Server

- **Auth0** — 가장 보편
- **Okta** — 엔터프라이즈
- **AWS Cognito**
- **Firebase Auth**
- **Supabase Auth**
- **Keycloak** — Open source
- **Google / Facebook / GitHub / Microsoft** — Social login

---

## 15. 흔한 함정

### 함정 1 — Authorization Code 만으로 안전 가정
PKCE 필수 (모바일 / SPA).

### 함정 2 — Redirect URI 검증 부족
공격자 URI 등록 → token 도난.

### 함정 3 — Refresh Token 의 보관
HttpOnly Cookie 또는 secure storage. localStorage X.

### 함정 4 — State 파라미터 누락
CSRF 위험.

### 함정 5 — alg:none JWT
서버가 알고리즘 검증.

### 함정 6 — Scope 너무 광범위
"모든 권한 요청" — 사용자가 거부 / 신뢰 ↓. 최소 권한.

### 함정 7 — Implicit Flow 사용
deprecated — PKCE.

### 함정 8 — Logout 의 OIDC end_session
서버 세션도 종료 필요 (RP-Initiated Logout).

---

## 16. 학습 자료

- **RFC 6749** (OAuth 2.0)
- **RFC 7636** (PKCE)
- **RFC 8628** (Device Code)
- **RFC 9068** (OIDC)
- **OAuth 2.1 draft**
- "OAuth 2.0 Simplified" — Aaron Parecki
- Auth0 / Okta 문서

---

## 17. 관련

- [[auth]] — Auth hub
- [[bearer-token]] — Access Token (JWT)
- [[../cookies/jwt-vs-cookie]]
- [[../../../security-theory/security-theory]]
