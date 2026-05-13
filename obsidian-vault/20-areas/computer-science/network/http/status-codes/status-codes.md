---
title: "HTTP 상태 코드 (Hub)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T21:10:00+09:00
tags:
  - network
  - http
  - status-codes
---

# HTTP 상태 코드 (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 5 카테고리 + 자주 쓰는 코드 인덱스 |

**[[../http|↑ HTTP]]** · **[[../../network|↑↑ network hub]]**

---

## 1. 5 카테고리

| 범위 | 의미 | 노트 |
| --- | --- | --- |
| **1xx** | Informational — 처리 중 | [[1xx-informational]] |
| **2xx** | Success — 정상 처리 | [[2xx-success]] |
| **3xx** | Redirection — 추가 동작 필요 | [[3xx-redirection]] |
| **4xx** | Client Error — 클라이언트 오류 | [[4xx-client-errors]] |
| **5xx** | Server Error — 서버 오류 | [[5xx-server-errors]] |

---

## 2. 가장 자주 쓰는 30 개

### 2xx
- **200 OK** — 정상 응답
- **201 Created** — 자원 생성 (POST)
- **202 Accepted** — 비동기 처리 시작
- **204 No Content** — body X
- **206 Partial Content** — Range 응답

### 3xx
- **301 Moved Permanently** — 영구 이전
- **302 Found** — 임시 이전 (역사적)
- **304 Not Modified** — 캐시 valid
- **307 Temporary Redirect** — 메서드 보존
- **308 Permanent Redirect** — 메서드 보존

### 4xx
- **400 Bad Request** — 잘못된 요청
- **401 Unauthorized** — 인증 필요
- **403 Forbidden** — 인가 거부
- **404 Not Found** — 자원 없음
- **405 Method Not Allowed**
- **406 Not Acceptable** — 협상 실패
- **408 Request Timeout**
- **409 Conflict** — 충돌
- **410 Gone** — 영구 삭제
- **413 Payload Too Large**
- **415 Unsupported Media Type**
- **422 Unprocessable Entity** — 의미 오류
- **429 Too Many Requests** — Rate limit

### 5xx
- **500 Internal Server Error**
- **501 Not Implemented**
- **502 Bad Gateway** — Proxy/LB 의 백엔드 오류
- **503 Service Unavailable** — 일시 불가
- **504 Gateway Timeout**
- **505 HTTP Version Not Supported**

---

## 3. 형식

```http
HTTP/1.1 200 OK
HTTP/1.1 404 Not Found
HTTP/2 200          (HTTP/2 는 reason phrase 없음)
```

### Status-Line
```
[HTTP-Version] SP [Status-Code] SP [Reason-Phrase] CRLF
```

- **Reason-Phrase** — 사람 가독 (RFC 9110 에선 옵션)
- HTTP/2/3 은 reason 제거 — 코드만

---

## 4. 분류 의도

### 1xx — 처리 중
- 클라가 더 기다려야 함
- 보통 자동 처리, 사용자 화면 X

### 2xx — 성공
- 요청 처리 OK
- body 가 자원 표현

### 3xx — 다른 자원 / 처리
- Redirect: Location 헤더의 새 URL
- 304: 캐시 사용

### 4xx — 클라 오류
- 요청 잘못 (body / URL / 인증 / 권한)
- 클라가 수정해서 재시도

### 5xx — 서버 오류
- 서버 문제 (코드 / 인프라 / 의존성)
- 클라가 시간 후 재시도

---

## 5. 코드 선택 가이드

### "성공인데 body 없음" → 204 No Content
```
DELETE /users/123 → 204
```

### "생성됨" → 201 Created + Location
```
POST /users → 201 Created
Location: /users/123
```

### "비동기 시작" → 202 Accepted
```
POST /jobs → 202 Accepted
Location: /jobs/abc/status
```

### "인증 X" → 401, "인가 X" → 403

| 401 | 403 |
| --- | --- |
| "넌 누구?" | "넌 알아, 권한 X" |
| WWW-Authenticate 헤더 | 권한 거부 |
| 토큰 보내라 | 토큰 OK, 권한 부족 |

### "없음" → 404, "있었지만 삭제" → 410

- 410 은 영구 — 검색엔진이 색인 제거
- 404 는 일반적

### "검증 실패" → 400 vs 422

- 400 — syntactic (JSON 형식 X)
- 422 — semantic (검증 실패, 비즈니스 규칙)

→ 응용 정책. 일부는 둘 다 400.

### "Rate limit" → 429
- `Retry-After` 헤더 추천

### "서버 잠시 안 됨" → 503
- `Retry-After` 헤더 권장
- 점검 / 과부하

---

## 6. 함정

### 함정 1 — 모든 오류를 500
RESTful 한 응용은 상세 코드 사용. 클라가 처리 방법 결정.

### 함정 2 — 모든 200 OK + JSON error
```
HTTP/1.1 200 OK
{"error": "Not found"}    ← 안티패턴
```

→ 404 Not Found + JSON error body. HTTP 의미 활용.

### 함정 3 — 200 vs 201 vs 204
POST 응답:
- body 있음 + 생성됨 → 201
- body 없음 → 204
- 처리만 (자원 생성 X) → 200

### 함정 4 — 302 의 메서드 처리
역사적: 302 후 GET 으로 — RFC 위반. 명확히: **307 / 308** (메서드 보존).

### 함정 5 — 401 vs 403 혼동
대부분 401. "인증 후 권한 부족" 만 403.

### 함정 6 — Reason Phrase 의존
HTTP/2+ 는 없음. 코드만 신뢰.

### 함정 7 — 비표준 코드
418 (I'm a teapot) — RFC 2324 의 농담 코드. 사용 X.
Cloudflare 의 520-526 — vendor 특수.

---

## 7. RFC 표준 코드 (RFC 9110 Section 15)

전체 표준 코드:
- 1xx: 100, 101, 102, 103
- 2xx: 200-208, 226
- 3xx: 300-308
- 4xx: 400-418, 421-426, 428, 429, 431, 451
- 5xx: 500-511

비표준 / 벤더:
- 418 (RFC 2324 농담)
- 419-499 (벤더 / 미할당)
- 520-526 (Cloudflare)
- 598 (Network read timeout - 비표준)

---

## 8. 학습 자료

- **RFC 9110** Section 15
- MDN HTTP status — https://developer.mozilla.org/en-US/docs/Web/HTTP/Status
- "REST API status codes" 가이드
- IANA Status Code Registry

---

## 9. 관련

- [[../http]] — HTTP hub
- [[../methods/methods]] — 메서드와 함께
- [[../headers/headers]] — Location, Retry-After, WWW-Authenticate
- 카테고리별 — [[1xx-informational]] [[2xx-success]] [[3xx-redirection]] [[4xx-client-errors]] [[5xx-server-errors]]
