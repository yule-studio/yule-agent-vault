---
title: "토큰 모델 — JWT vs Session + 서명 알고리즘"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:15:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - design-decisions
  - jwt
  - session
---

# 토큰 모델 — JWT vs Session + 서명 알고리즘

**[[design-decisions|↑ design-decisions hub]]**

> "사용자 인증 상태를 어떻게 표현 / 검증하나" — JWT (stateless) 또는 Session (stateful). 잘못 선택하면 **MSA 확장 불가 또는 즉시 무효화 불가**.

---

## 1. 본 vault 결정

- **JWT 표준** (HS256, access 15분 + opaque refresh 14일).
- access 짧게 → 무효화 부담 ↓.
- refresh 는 opaque + 서버 매핑 → 즉시 무효 가능 (DB delete).

---

## 2. JWT vs Session — 4구조

### 2.1 JWT (Stateless)

**왜 적합한 케이스**
- MSA — 각 서비스가 독립적으로 토큰 검증 (서명만 확인, DB 호출 X).
- 분산 환경 — 클라가 토큰 들고 다님, 서버는 stateless.
- 외부 / 모바일 클라 — Cookie 의존 X.

**왜 안 되는 케이스 (Session 이 더 나음)**
- 즉시 무효화 필요한 환경 (관리자 강제 logout 즉시 반영).
- 토큰에 큰 데이터 (권한 목록 100개+) → JWT 크기 폭증.

**안 하면 무슨 문제 (JWT 만으로 안 됨)**
- 만료까지 토큰 사용 가능 — 분실 / 도난 시 영향 시간 ↑.
- 권한 변경 즉시 반영 안 됨.

**대안과 왜 안 됨**
- Session — 매 요청 DB 호출 → MSA 확장 X. sticky session 필요.

**트레이드오프**
- 즉시 무효화 X → **opaque refresh + 짧은 access** 로 보완 (본 vault).
- 검증 비용 ↓ (서명만) — 매 API 호출마다 ms.

---

### 2.2 Session (Stateful)

**왜 적합한 케이스**
- 모놀리식 단일 서비스 — Redis / DB 한 곳에서 session 관리.
- 강한 무효화 — admin 이 user 강제 로그아웃 즉시.
- 세션에 큰 state (장바구니 / 진행 중 작업).

**왜 안 되는 케이스**
- MSA — 각 서비스가 session store 공유 → 부담 + latency.
- 모바일 / API — Cookie 의존 어색.

**안 하면 무슨 문제 (Session 만 사용 시)**
- 분산 환경 확장 어려움 — session store 가 SPOF.
- 매 API 호출 DB / Redis lookup → latency.

**대안**
- JWT + 자체 server blacklist (Redis) — Session 이점 + JWT 의 stateless 성. 본 vault 의 hybrid 접근.

---

### 2.3 본 vault — Hybrid (JWT + opaque refresh)

```
[Login]
  ← access JWT (15분, HS256)
  ← refresh opaque (14일, DB 저장)

[API call]
  Authorization: Bearer <access>
  → JWT 서명 검증만 (stateless)

[Refresh]
  POST /auth/refresh   Cookie: refresh=<opaque>
  → DB 조회 + ROTATED 검증 (stateful)
  ← 새 access + 새 refresh

[Logout]
  → DB 의 refresh 삭제
  → 옛 access 는 15분 후 자연 만료
```

**왜 이 hybrid**
- access JWT = stateless, 빠른 검증, 매 API 호출.
- refresh opaque = stateful, 즉시 무효 가능 (DB delete).
- access 짧음 (15분) = 무효화 안 해도 큰 영향 X.
- refresh rotation + reuse detection = 도난 감지.

**왜 access 가 15분 (1시간 / 1일 아님)**
- 짧을수록 무효화 부담 ↓ — 도난 시 영향 시간 ↓.
- 너무 짧으면 (1분) — refresh 폭증 → DB 부담.
- 15분 = 산업 표준 (Google, Twitter).

**왜 refresh 가 14일 (30일 / 90일 아님)**
- 너무 길면 도난 시 피해 ↑.
- 너무 짧으면 사용자 잦은 재로그인 → UX 망함.
- 14일 + rotation = 활성 사용자는 사실상 무기한, 비활성은 자연 만료.

---

## 3. JWT 서명 알고리즘 — HS256 vs RS256 vs ES256

### 3.1 HS256 (대칭 키)

**왜 적합**
- 모놀리식 / 단일 서비스 — 발급자 = 검증자.
- 단일 secret (대칭) 으로 발급 / 검증.
- 빠름 (서명 / 검증 모두 ms).

**왜 안 됨 (MSA)**
- 발급자 / 검증자가 다르면 secret 공유 필요 → 보안 risk.
- secret 유출 시 모든 서비스 영향.

**안 하면 무슨 문제**
- single secret rotation 시 모든 인스턴스 동시 갱신 필요.
- secret 분실 시 모든 발급 토큰 invalidate.

**트레이드오프**
- 단순 / 빠름 vs MSA 확장 어려움.
- 본 vault 의 default (모놀리식 가정).

---

### 3.2 RS256 (비대칭 키)

**왜 적합**
- MSA — IdP 가 private key 로 발급, 각 서비스가 public key 로 검증.
- 외부 검증자 (third-party API) — public key 만 공개 (JWKS endpoint).
- key rotation 시 — public key 만 업데이트 (private key 보호).

**왜 안 됨 (모놀리식)**
- HS256 대비 검증 비용 ↑ (수 ms).
- 키 관리 복잡 (private + public).

**대안과 왜 안 됨**
- ES256 — RS256 보다 빠름 (ECDSA) + 작은 키. 대안 OK.

**트레이드오프**
- key 관리 부담 ↑ vs MSA 확장성.

---

### 3.3 ES256 (ECDSA P-256)

**왜 적합**
- RS256 의 비대칭 이점 + 작은 키 / 빠른 검증.
- 모바일 / 임베디드 환경에 유리.

**왜 잘 안 씀 (한국 SaaS)**
- 인지도 / 도구 지원이 RS256 보다 적음.
- 한국 IdP / 카카오 / 네이버는 RS256 표준.

**언제 고려**
- 글로벌 / 모바일 우선 — ES256.
- 한국 표준 따를 시 — RS256.

---

### 3.4 `none` / weak HS256

**왜 절대 X**
- `alg: "none"` — 검증 우회 (역사적 사고: 2015 jwt 라이브러리 취약점 다수).
- 약한 secret (`"secret"`, `"123456"`) — brute force 1초 안에.

**방어**
- `JwtParser.requireSigned(true)`.
- 라이브러리가 `alg: "none"` reject (최신 jjwt 는 default).
- secret 은 256 bit random.

---

## 4. JWT claim 구조

```json
{
  "iss": "yule-studio",
  "sub": "01HQXY7K3FAB2CDEFGHIJKMNPQR",
  "iat": 1715731200,
  "exp": 1715732100,
  "jti": "01HQAUTHTOKEN001",
  "role": "USER",
  "sid": "01HQDEVICE001"
}
```

| claim | 의미 | 왜 |
| --- | --- | --- |
| iss | 발급자 | 다중 IdP 환경에서 출처 확인 |
| sub | user id (ULID) | 표준 — 인증 대상 |
| iat | 발급 시각 | clock skew / replay 감지 |
| exp | 만료 시각 | 강제 만료 |
| jti | JWT id | revoke / audit |
| role | 권한 | 매 요청마다 DB 조회 회피 (15분 stale OK) |
| sid | session / device id | refresh 연결 |

**왜 권한 (role) 을 JWT 안에**
- 매 API 호출마다 DB 의 user.role 조회 → 부담.
- 15분 stale OK (강등 시 강제 logout 정책으로 보완).

**왜 user.email / name 을 JWT 안에 X**
- 변경 가능 — stale 가능.
- 매 응답에 user 정보 필요하면 별도 `/me` endpoint.

---

## 5. JWT vs Cookie vs LocalStorage

| 저장 위치 | XSS 위협 | CSRF 위협 | 사용처 |
| --- | --- | --- | --- |
| **HttpOnly Cookie** | ✅ JS 접근 X | ⚠️ CSRF token 필요 | 웹 표준 |
| **LocalStorage** | ❌ XSS 시 즉시 도난 | ✅ same-origin | 모바일 / SPA |
| **Memory (in-app)** | ⚠️ 페이지 새로고침 시 사라짐 | ✅ | 보안 우선 SPA |

**본 vault**
- access JWT: Authorization header (`Bearer ...`) — 매 API 호출에 명시.
- refresh: HttpOnly + Secure + SameSite=Strict Cookie — XSS 방어 + CSRF token 같이.

**왜 refresh 만 Cookie**
- refresh 가 access 보다 민감 (14일 유효).
- XSS 노출되면 영향 큼 → HttpOnly.

**왜 access 는 header**
- 매 API 호출에 명시 — CSRF 자연스럽게 방어 (header 가 same-origin only).
- 모바일 / API client 와 호환.

---

## 6. JWT 라이브러리

```kotlin
implementation("io.jsonwebtoken:jjwt-api:0.12.6")
runtimeOnly("io.jsonwebtoken:jjwt-impl:0.12.6")
runtimeOnly("io.jsonwebtoken:jjwt-jackson:0.12.6")
```

**왜 jjwt**
- 한국 / 글로벌 Spring Boot 표준.
- 0.12+ 는 fluent API 안정화.
- `alg: "none"` 거부 기본.

**대안**
- Auth0 `java-jwt` — 비슷.
- Nimbus `nimbus-jose-jwt` — 더 강력 (JWE / JWS 모두).

---

## 7. 함정 모음

### 함정 1 — JWT 만 + 만료 너무 김 (24h+)
도난 시 24시간 사용 가능. 즉시 무효 불가.
→ access 짧게 (15분) + opaque refresh.

### 함정 2 — refresh 도 JWT 로
무효화 어려움. 도난 시 14일 유효.
→ refresh 는 opaque (random string) + DB 저장.

### 함정 3 — JWT 에 secret 정보 (password / phone)
JWT 는 서명만, **암호화 X** (base64 디코드 가능).
→ 민감 정보 절대 안 넣음.

### 함정 4 — Authorization 헤더에 refresh
매 API 호출에 refresh 노출 → 도난 위험 ↑.
→ refresh 는 cookie only.

### 함정 5 — `alg: "none"` 라이브러리 취약점
2015 사고 다수. 최신 jjwt 는 기본 거부.
→ 라이브러리 버전 fix + `requireSigned(true)`.

### 함정 6 — secret 약함 (`"secret"`)
1초 brute force.
→ 256-bit random secret (`openssl rand -base64 32`).

### 함정 7 — secret 코드에 hardcoded
git 유출 = 즉시 사고.
→ env / KMS / secret manager.

### 함정 8 — clock skew 대응 안 함
서버 간 시계 차이 (1초+) 로 토큰 invalid.
→ `allowedClockSkewSeconds(30)`.

### 함정 9 — refresh rotation 안 함
도난 시 14일 사용 가능.
→ 매 사용 시 새 refresh 발급 + 옛 것 ROTATED.

### 함정 10 — Reuse detection 안 함
ROTATED 의 refresh 재사용 = 도난 신호인데 무시.
→ chain 전체 revoke + user 알람.

### 함정 11 — JWT 안에 너무 많은 데이터
크기 폭증 → 매 API 호출에 큰 헤더 → 대역폭 / latency ↑.
→ 핵심만 (sub, role, sid). 나머지는 별도 fetch.

### 함정 12 — Token blacklist 없음 (강제 logout 필요)
admin 강등 / 보안 사고 시 즉시 무효화 X.
→ Redis blacklist 또는 access 매우 짧게 (15분).

---

## 8. 다른 컨텍스트

### 8.1 MSA

```yaml
token-model:
  jwt-algorithm: RS256
  jwks-endpoint: https://idp.example.com/.well-known/jwks.json
  access-ttl: 15m
  reason: 각 서비스 독립 검증 / public key 만 공유
```

### 8.2 모바일 우선

```yaml
token-model:
  jwt-algorithm: HS256
  refresh-storage: secure-enclave / keychain
  access-ttl: 1h               # 모바일은 1h OK (네트워크 비용)
```

### 8.3 금융 / 의료

```yaml
token-model:
  jwt-algorithm: RS256
  access-ttl: 5m              # 매우 짧게
  refresh-ttl: 1h             # 매우 짧게
  step-up: required for sensitive ops
  reason: 강한 보안 + 즉시 무효화
```

---

## 9. 관련

- [[design-decisions|↑ hub]]
- [[refresh-storage]] — refresh 어디에 저장
- [[../database/refresh-tokens-table]] — refresh 의 schema
- [[../security]] — 도난 대응
- [[../login-impl]] · [[../token-refresh-impl]]
- 외부 — RFC 7519 (JWT), OWASP JWT Cheat Sheet
