---
title: "Response Headers — 응답 전용"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T21:50:00+09:00
tags:
  - network
  - http
  - headers
  - response
---

# Response Headers — 응답 전용

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Server / Location / WWW-Authenticate / Set-Cookie / Allow / Retry-After / ETag |

**[[headers|↑ Headers]]** · **[[../http|↑↑ HTTP]]**

---

## 1. Server

```http
Server: nginx/1.25.0
Server: Apache/2.4.54 (Ubuntu)
Server: cloudflare
Server: Microsoft-IIS/10.0
```

### 의미
- 서버 소프트웨어 식별

### 보안 권장
- **버전 노출 X** — 취약점 스캔 표적
- 일부 환경: 헤더 제거 / 일반화

```nginx
server_tokens off;        # Nginx — 버전 숨김

# 더 강하게 — header_filter_module 으로 완전 제거
more_clear_headers Server;
```

### X-Powered-By (유사)

```http
X-Powered-By: PHP/8.0.0
X-Powered-By: Express
```

- 비표준이지만 보편
- **반드시 제거** — 정보 누출

```javascript
// Express
app.disable('x-powered-by');
```

---

## 2. Location

```http
HTTP/1.1 301 Moved Permanently
Location: https://example.com/new-path

HTTP/1.1 201 Created
Location: /api/users/123
```

### 사용
- **3xx Redirect** — 새 URL
- **201 Created** — 생성된 자원 URI
- **202 Accepted** — 비동기 작업의 status URL

### 절대 vs 상대
- 절대: `https://example.com/path`
- 상대: `/path` 또는 `./path`
- RFC 9110 — 둘 다 OK, 절대 권장

---

## 3. WWW-Authenticate

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm="API"
WWW-Authenticate: Bearer realm="API", error="invalid_token"
WWW-Authenticate: Digest realm="API", nonce="abc", qop="auth"
```

### 의미
- 401 응답의 **인증 방식** 알림
- 클라이언트 / 브라우저가 인증 정보 수집

### Bearer 의 오류 정보

```http
WWW-Authenticate: Bearer realm="API", error="invalid_token",
    error_description="The access token expired"
```

OAuth 2.0 의 RFC 6750.

자세히 → [[../auth/auth]]

---

## 4. Proxy-Authenticate

```http
HTTP/1.1 407 Proxy Authentication Required
Proxy-Authenticate: Basic realm="proxy"
```

- WWW-Authenticate 의 proxy 버전
- 407 응답에

---

## 5. Set-Cookie

```http
Set-Cookie: sessionid=abc123; Path=/; HttpOnly; Secure; SameSite=Strict; Max-Age=3600
Set-Cookie: theme=dark; Path=/; Max-Age=31536000
```

### 속성
- **Path / Domain** — 범위
- **Expires / Max-Age** — 수명
- **Secure / HttpOnly / SameSite / Partitioned** — 보안

자세히 → [[../cookies/cookies]]

### 중요
- 한 응답에 여러 `Set-Cookie` 가능 (별도 헤더로)
- 브라우저가 후속 요청에 자동 첨부

---

## 6. Allow

```http
HTTP/1.1 405 Method Not Allowed
Allow: GET, POST, OPTIONS
```

- 405 응답에 **필수** (RFC 9110)
- 200 응답에도 가능 (OPTIONS 등)

---

## 7. Retry-After

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60                                  (60 초)

HTTP/1.1 503 Service Unavailable
Retry-After: Tue, 13 May 2026 14:00:00 GMT       (HTTP date)
```

### 사용
- 429 (rate limit)
- 503 (service unavailable)
- 일부 3xx (점진 redirect)

### 형식
- 정수 (초) — 가장 흔함
- HTTP date — 절대 시점

---

## 8. ETag

```http
HTTP/1.1 200 OK
ETag: "v1.0-abc123"
ETag: W/"v1.0-abc123"          (weak)
```

### 의미
- 자원의 fingerprint (해시 / 버전)
- 조건부 GET / PUT 에 사용

### Strong vs Weak
- **Strong** `"abc"` — byte 단위 동일
- **Weak** `W/"abc"` — 의미 동일 (gzip 압축 차이 OK)

자세히 → [[../caching/etag-conditional]]

---

## 9. Last-Modified

```http
HTTP/1.1 200 OK
Last-Modified: Tue, 13 May 2026 10:00:00 GMT
```

- 자원의 마지막 수정 시간
- If-Modified-Since 의 짝

자세히 → [[../caching/last-modified]]

---

## 10. Vary

```http
HTTP/1.1 200 OK
Vary: Accept-Encoding, Accept-Language
```

### 의미
- 응답이 어느 요청 헤더에 따라 다른지
- 캐시 키의 일부

### 함정
- `Vary: *` — 캐시 X (모든 헤더에 따라 다름)
- 너무 많은 Vary — 캐시 효율 ↓
- `Vary: User-Agent` — 매우 비효율 (UA 종류 많음)

자세히 → [[../caching/vary-header]]

---

## 11. Content-* (응답 — Entity)

```http
Content-Type: application/json; charset=UTF-8
Content-Length: 134
Content-Encoding: gzip
Content-Language: ko-KR
Content-Disposition: attachment; filename="report.pdf"
Content-Range: bytes 1000-1999/5000
Content-Location: /users/123.json
```

자세히 → [[entity-headers]]

---

## 12. Accept-Ranges

```http
Accept-Ranges: bytes
Accept-Ranges: none
```

- 자원이 Range request 지원하는지 알림
- 보통 정적 파일 (이미지, 비디오, 파일) 가 `bytes`

---

## 13. Age (캐시)

```http
HTTP/1.1 200 OK
Age: 300                                     (300 초 전 캐시됨)
```

- 캐시된 응답이 얼마나 오래됐는지
- CDN / Proxy 가 추가

---

## 14. Server-Timing (성능)

```http
HTTP/1.1 200 OK
Server-Timing: db;dur=53.2, cache;dur=2.1;desc="Redis", total;dur=58.7
```

### 의미
- 서버 측 처리 시간 분해
- 브라우저 DevTools 가 표시

### 사용
- Web Vitals 분석
- 응용 모니터링

---

## 15. Link

```http
Link: <https://example.com/next>; rel="next"
Link: </style.css>; rel="preload"; as="style"
Link: <https://api.com/users>; rel="profile"
```

### 사용
- HTTP Header 형식의 `<link>` 태그
- Pagination (next, prev, first, last)
- Resource hints (preload, prefetch, preconnect, dns-prefetch)
- 103 Early Hints

---

## 16. Alt-Svc

```http
HTTP/1.1 200 OK
Alt-Svc: h3=":443"; ma=86400
```

- HTTP/3 알림 — "다음부터 HTTP/3 시도"
- max-age 캐시

자세히 → [[../versions/version-comparison]]

---

## 17. 보안 헤더 (응답)

```http
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
Content-Security-Policy: default-src 'self'
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(), microphone=()
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
Cross-Origin-Resource-Policy: same-origin
```

자세히 → [[../security/security]]

---

## 18. CORS (응답)

```http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 86400
Access-Control-Expose-Headers: X-Total-Count
```

자세히 → [[../cors/cors]]

---

## 19. 함정

### 함정 1 — Server / X-Powered-By 노출
보안 — 제거 권장.

### 함정 2 — Location 의 invalid URL
브라우저 무한 redirect.

### 함정 3 — Set-Cookie 의 SameSite=None 누락
크로스 사이트 쿠키 거부. Secure 도 같이.

### 함정 4 — Retry-After 누락
429/503 만 보내면 클라가 무차별 retry.

### 함정 5 — ETag 의 weak/strong 혼동
PUT If-Match 는 strong 만 (RFC).

### 함정 6 — Vary: *
캐시 무효화 — 의도 외.

### 함정 7 — Allow 누락 (405)
RFC 위반.

---

## 20. 학습 자료

- **RFC 9110** Section 8, 10
- **RFC 6750** (OAuth 2.0 Bearer)
- MDN Response headers

---

## 21. 관련

- [[headers]] — Headers hub
- [[request-headers]] — 짝
- [[entity-headers]] — Content-*
- [[../caching/etag-conditional]]
- [[../caching/vary-header]]
- [[../cookies/cookies]]
- [[../security/security]]
- [[../cors/cors]]
