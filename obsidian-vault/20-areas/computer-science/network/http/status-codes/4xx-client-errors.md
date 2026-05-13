---
title: "4xx — Client Errors"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T21:25:00+09:00
tags:
  - network
  - http
  - status-codes
  - 4xx
---

# 4xx — Client Errors

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 400-431 + 451 |

**[[status-codes|↑ Status Codes]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

"클라이언트의 잘못된 요청" — 요청 형식 / 권한 / 자원 등 클라가 수정해서 재시도.

---

## 2. 표준 코드

### 400 Bad Request

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json
{"error":"Invalid JSON"}
```

#### 사용
- 잘못된 JSON / XML
- 필수 필드 누락
- 잘못된 헤더
- 너무 모호 — 명확한 4xx 가 있으면 그것

### 401 Unauthorized

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer realm="api"
```

#### 의미 — "넌 누구?"
- 인증 자격 누락 / 잘못
- **이름이 잘못됨** — RFC: "Unauthenticated" 가 정확

#### WWW-Authenticate 헤더 필수
```http
WWW-Authenticate: Basic realm="..."
WWW-Authenticate: Bearer realm="api"
WWW-Authenticate: Digest realm="..." nonce="..."
```

### 402 Payment Required (reserved)
RFC 가 정의했지만 실용 X. Stripe 등 일부 응용 — "구독 만료" 등.

### 403 Forbidden

```http
HTTP/1.1 403 Forbidden
Content-Type: application/json
{"error":"Insufficient permissions"}
```

#### 의미 — "넌 알아, 권한 X"
- 인증 OK, 인가 거부
- 토큰 valid 하지만 admin 만 가능
- IP 차단

#### 401 vs 403

| 측면 | 401 | 403 |
| --- | --- | --- |
| 인증 | 실패 | 성공 |
| 의미 | "넌 누구?" | "권한 X" |
| WWW-Authenticate | 필수 | 선택 |
| 재시도 | 다른 자격 시도 | 자격 다시 받아도 X |

### 404 Not Found

```http
HTTP/1.1 404 Not Found
```

#### 의미
- 자원 없음
- URL 잘못
- 일부 시스템 — "존재하지만 권한 X" 도 (403 노출 회피)

#### 404 vs 410
- 404 — "없음" (영구 / 임시 불명)
- 410 — "영구 삭제 / 이전 X" (검색엔진 색인 제거)

### 405 Method Not Allowed

```http
GET /api/users HTTP/1.1   (POST만 허용)

HTTP/1.1 405 Method Not Allowed
Allow: POST, OPTIONS
```

- **Allow 헤더 필수** (허용 메서드)

### 406 Not Acceptable

```http
GET /api/users HTTP/1.1
Accept: application/xml

HTTP/1.1 406 Not Acceptable
(서버가 XML 못 만들고 JSON 만 가능)
```

- 클라가 받을 수 있는 표현 X
- Content Negotiation 실패

### 407 Proxy Authentication Required

```http
HTTP/1.1 407 Proxy Authentication Required
Proxy-Authenticate: Basic realm="proxy"
```

- 401 의 proxy 버전
- HTTP proxy 인증 필요

### 408 Request Timeout

```http
HTTP/1.1 408 Request Timeout
```

- 클라가 요청 안 보내고 idle
- 서버가 timeout 후 종료

### 409 Conflict

```http
HTTP/1.1 409 Conflict
{"error":"Username already taken"}
```

- 자원 상태 충돌
- 동시 갱신 (낙관적 잠금 실패) — `If-Match` 실패는 412 도 가능
- 중복 생성 (unique 위반)

### 410 Gone

```http
HTTP/1.1 410 Gone
```

- **영구 삭제 / 이전 없이 제거**
- 404 보다 강한 의미
- 검색엔진: 색인 제거 (404 도 결국 제거하지만 410 이 빠름)

### 411 Length Required

```http
POST /api/upload HTTP/1.1
(Content-Length 누락)

HTTP/1.1 411 Length Required
```

- Content-Length 헤더 누락 (POST/PUT)
- 일부 서버 / 옛 HTTP/1.0 의 잔재

### 412 Precondition Failed

```http
PUT /users/123 HTTP/1.1
If-Match: "v1"

HTTP/1.1 412 Precondition Failed
```

- If-Match / If-Unmodified-Since 실패
- 낙관적 잠금

### 413 Payload Too Large (Content Too Large)

```http
POST /api/upload HTTP/1.1
Content-Length: 10737418240

HTTP/1.1 413 Content Too Large
Retry-After: 0
```

- 본문 너무 큼
- Nginx `client_max_body_size`, Apache `LimitRequestBody`

### 414 URI Too Long

```http
GET /api/search?q=...(매우 김)... HTTP/1.1

HTTP/1.1 414 URI Too Long
```

- URL 너무 김
- 보통 GET body 못 보내 query string 비대 → POST 권장

### 415 Unsupported Media Type

```http
POST /api/users HTTP/1.1
Content-Type: application/xml

HTTP/1.1 415 Unsupported Media Type
Accept-Post: application/json
```

- Content-Type 미지원
- Accept-Post 헤더로 지원 타입 알림 (response)

### 416 Range Not Satisfiable

```http
GET /file HTTP/1.1
Range: bytes=99999-

HTTP/1.1 416 Range Not Satisfiable
Content-Range: bytes */5000
```

- Range 가 파일 크기 초과
- Content-Range `*/total` 로 실제 크기 알림

### 417 Expectation Failed

```http
PUT /upload HTTP/1.1
Expect: 100-continue

HTTP/1.1 417 Expectation Failed
```

- Expect 헤더 충족 불가

### 418 I'm a teapot
RFC 2324 농담 (4월 1일). 실용 X.

### 421 Misdirected Request
HTTP/2 의 Connection Coalescing 실패. 클라가 다시 연결.

### 422 Unprocessable Content (WebDAV → RFC 9110)

```http
POST /api/users HTTP/1.1
{"email":"not-an-email"}

HTTP/1.1 422 Unprocessable Content
{"errors":[{"field":"email","message":"Invalid format"}]}
```

#### 400 vs 422
- 400 — syntactic (JSON 형식 X)
- 422 — semantic (검증 / 비즈니스 규칙)
- 응용 정책. Rails 표준 — 422.

### 423 Locked (WebDAV)
자원 잠김.

### 424 Failed Dependency (WebDAV)
선행 작업 실패 → 종속 작업 거부.

### 425 Too Early (RFC 8470)
0-RTT 재전송 위험 거부 (TLS 1.3 / QUIC).

### 426 Upgrade Required

```http
HTTP/1.1 426 Upgrade Required
Upgrade: TLS/1.2, HTTP/1.1
Connection: Upgrade
```

- HTTPS / HTTP/2 강제.

### 428 Precondition Required

```http
HTTP/1.1 428 Precondition Required
```

- "조건부 요청 (If-Match) 으로 보내" — Lost Update 방지

### 429 Too Many Requests

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1715587200
```

- Rate Limit
- Retry-After 헤더 표준 (초 또는 HTTP date)
- X-RateLimit-* — 비표준이지만 보편

### 431 Request Header Fields Too Large

```http
HTTP/1.1 431 Request Header Fields Too Large
```

- 헤더 전체 / 개별 너무 큼
- Cookie 과다 / referrer 큰 URL

### 451 Unavailable For Legal Reasons

```http
HTTP/1.1 451 Unavailable For Legal Reasons
Link: <https://example.com/legal>; rel="blocked-by"
```

- 법적 차단 (검열, 저작권 등)
- 451 = Fahrenheit 451 (책 검열 소설)

---

## 3. Validation Error 의 응답 형식

### RFC 7807 — Problem Details for HTTP APIs

```http
HTTP/1.1 422 Unprocessable Content
Content-Type: application/problem+json

{
  "type": "https://example.com/probs/validation",
  "title": "Validation Failed",
  "status": 422,
  "detail": "Email field is required",
  "instance": "/api/users",
  "errors": [
    {"field":"email","code":"required"},
    {"field":"age","code":"min","value":0}
  ]
}
```

→ 표준 — 일관된 에러 응답.

---

## 4. 함정

### 함정 1 — 모든 4xx 를 400
RESTful 응용은 401/403/404/409/422/429 분리 — 클라 처리 결정.

### 함정 2 — 401 + 403 혼동
인증 = 401, 인가 = 403. 잘못 쓰면 클라가 잘못된 재시도.

### 함정 3 — 404 로 권한 노출 차단
"존재 자체 비밀" → 403 대신 404 — 일부 디자인. Spotify / GitHub 비공개 저장소.

### 함정 4 — 429 의 Retry-After 누락
클라가 재시도 시점 모름 — 무차별 retry.

### 함정 5 — 405 의 Allow 누락
RFC 위반. Allow 헤더 필수.

### 함정 6 — 422 vs 400 일관성
같은 검증 오류를 일관되게.

### 함정 7 — 413 / 414 / 431 의 server limit
Nginx / Apache 의 기본값으로 거부 — 응용 알기 어려움.

### 함정 8 — 451 의 메시지
`Link: rel="blocked-by"` 로 차단 주체 명시 — 투명성.

---

## 5. Rate Limit 헤더 (관례)

```http
X-RateLimit-Limit: 100              ← 시간당 100
X-RateLimit-Remaining: 75           ← 남은
X-RateLimit-Reset: 1715587200       ← Unix timestamp (재충전)
Retry-After: 60                     ← 또는 초
```

GitHub, Twitter, Stripe 가 사용. **RFC 표준 X** — `RateLimit-Limit` 등 표준화 작업 중.

---

## 6. 학습 자료

- **RFC 9110** Section 15.5
- **RFC 7807** (Problem Details)
- **RFC 7725** (451)
- **RFC 6585** (428/429/431/511)
- MDN HTTP 4xx

---

## 7. 관련

- [[status-codes]] — hub
- [[5xx-server-errors]] — 서버 오류
- [[../headers/response-headers]] — WWW-Authenticate, Retry-After, Allow
- [[../auth/auth]] — 401/403 의 인증
- [[../security/security]] — 451 검열
