---
title: "CORS + Credentials (Cookie / Authorization)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T23:25:00+09:00
tags:
  - network
  - http
  - cors
  - credentials
---

# CORS + Credentials (Cookie / Authorization)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | credentials: include + 함정 |

**[[cors|↑ CORS]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

Cross-origin 요청에 **Cookie / Authorization 헤더 / TLS client cert** 첨부하려면
특별한 CORS 헤더 + 클라 설정 필요.

---

## 2. 클라 설정

### fetch
```javascript
fetch('https://api.example.com/me', {
    credentials: 'include'    // 핵심
});
```

### XHR
```javascript
const xhr = new XMLHttpRequest();
xhr.withCredentials = true;
xhr.open('GET', 'https://api.example.com/me');
xhr.send();
```

### axios
```javascript
axios.get('https://api.example.com/me', {
    withCredentials: true
});
```

### credentials 값 (fetch)
- **omit** — 첨부 X (기본)
- **same-origin** — 같은 origin 만
- **include** — cross-origin 도 첨부

---

## 3. 서버 설정 (응답)

```http
Access-Control-Allow-Origin: https://app.example.com    ← 명시 (wildcard X)
Access-Control-Allow-Credentials: true
```

→ **둘 다 필요**.

### Preflight 도 동일

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 86400
```

---

## 4. Wildcard 의 금지

```http
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
```

→ **브라우저가 거부**.

### 이유
- `*` 면 모든 사이트가 사용자 cookie 첨부 → 보안 사고
- 명시적 origin 만 — "이 origin 은 신뢰"

### 해결 — 동적 origin

```python
origin = request.headers.get('Origin')
if origin in ALLOWED_ORIGINS:
    response.headers['Access-Control-Allow-Origin'] = origin
    response.headers['Access-Control-Allow-Credentials'] = 'true'
    response.headers['Vary'] = 'Origin'    ← 캐시
```

---

## 5. Vary: Origin 필수

```http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
Vary: Origin
```

### 이유
- 여러 origin 응답 가능 — 캐시가 분리
- 없으면 한 origin 의 응답이 다른 origin 에 잘못 캐시

---

## 6. 다른 Allow-Headers 도 명시

`*` 은 credentials 시 적용 X:

```http
Access-Control-Allow-Headers: Content-Type, Authorization, X-CSRF-Token
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
```

→ 모든 헤더 / 메서드 명시 필요.

### Expose-Headers

```http
Access-Control-Expose-Headers: X-Total-Count, X-Request-ID
```

- JS 가 읽을 수 있는 응답 헤더 (기본 7 개 외)
- 기본 노출 — Cache-Control, Content-Language, Content-Type, Expires, Last-Modified, Pragma
- 그 외는 명시

---

## 7. SameSite Cookie 와의 관계

```http
Set-Cookie: sessionid=abc; SameSite=None; Secure
```

### Cross-site 요청에 cookie 첨부 조건

1. **SameSite=None** + **Secure** (모던 브라우저)
2. **CORS** 의 Allow-Credentials: true + Allow-Origin (명시)
3. fetch 의 `credentials: 'include'`

### 모두 OK 면 → 사용자 cookie 자동 첨부

---

## 8. 전체 예 — JWT in Cookie SPA

### 서버

```python
# 로그인 응답
response.set_cookie(
    "__Host-token", jwt,
    httponly=True,
    secure=True,
    samesite="None",
    max_age=3600
)

# CORS
response.headers["Access-Control-Allow-Origin"] = "https://app.example.com"
response.headers["Access-Control-Allow-Credentials"] = "true"
response.headers["Vary"] = "Origin"
```

### 클라

```javascript
// 로그인 후 모든 요청 자동 token 첨부
fetch('https://api.example.com/me', {
    credentials: 'include'
}).then(r => r.json());
```

→ XSS 방어 (HttpOnly) + CSRF 방어 (SameSite + CORS) + Cross-domain.

---

## 9. CORS 의 인증 응답 (401 / 403)

```http
HTTP/1.1 401 Unauthorized
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
WWW-Authenticate: Bearer realm="api"
{"error": "Invalid token"}
```

→ CORS 헤더가 응답에 있어야 브라우저가 401 도 JS 에 전달.

---

## 10. 함정

### 함정 1 — Wildcard + Credentials
```
Allow-Origin: * + Allow-Credentials: true
```
→ 브라우저 거부. 명시 origin.

### 함정 2 — credentials 안 보냄
- fetch 기본 `same-origin` — cross-origin 시 cookie X
- `credentials: 'include'` 명시

### 함정 3 — 서버가 Credentials 헤더 누락
브라우저가 응답 자체는 받지만 JS 에 안 전달.

### 함정 4 — Vary: Origin 누락
캐시가 한 origin 응답을 다른 origin 에 반환 → 보안 사고.

### 함정 5 — SameSite=None 의 Secure
브라우저 거부 (HTTP 면). HTTPS 필수.

### 함정 6 — Preflight 와 본 요청의 일관성
preflight 에 credentials true 면 본 요청도 동일.

### 함정 7 — XHR 의 withCredentials
fetch 와 다른 옵션 — XHR 코드는 다른 패턴.

### 함정 8 — Cross-origin form
HTML form submission 은 credentials 자동 (옛 동작) — CORS X.

---

## 11. CORS vs CSRF 비교

| 측면 | CORS | CSRF |
| --- | --- | --- |
| 보호 | 응답 read | 요청 자체 |
| 위치 | 응답 헤더 | 요청 검증 |
| 브라우저 | 브라우저만 | 서버 측 |
| 우회 | 서버는 그냥 요청 | SameSite Cookie + Token |

→ **CORS 는 read 방지 / CSRF 는 write 방지** — 다른 개념.

자세히 → [[../security/security]]

---

## 12. 학습 자료

- **Fetch standard** — Credentials
- MDN — Access-Control-Allow-Credentials
- "CORS with credentials" — web.dev

---

## 13. 관련

- [[cors]] — CORS hub
- [[preflight]]
- [[cors-troubleshooting]]
- [[../cookies/cookie-attributes]] — SameSite=None
- [[../cookies/jwt-vs-cookie]]
- [[../security/security]] — CSRF
