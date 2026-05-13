---
title: "Simple Request — Preflight 면제 조건"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T23:15:00+09:00
tags:
  - network
  - http
  - cors
  - simple-request
---

# Simple Request — Preflight 면제 조건

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Simple 조건 + 의미 + 함정 |

**[[cors|↑ CORS]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

CORS 의 **Preflight 가 면제되는** 요청 — 옛 HTML 폼이 만들 수 있던 요청과 동등한
형식. 브라우저가 직접 요청 + 응답 검증.

---

## 2. Simple Request 의 3 조건

모두 만족해야:

### 조건 1 — 메서드

- **GET**
- **HEAD**
- **POST**

→ PUT, DELETE, PATCH 는 X.

### 조건 2 — 헤더

다음만 허용 (CORS-safelisted request-header):
- **Accept**
- **Accept-Language**
- **Content-Language**
- **Content-Type** (제한 — 아래)
- 일부 더 (Range, ...)

→ **Authorization, X-API-Key, Custom 헤더** 면 preflight.

### 조건 3 — Content-Type

POST 에 한해 다음만:
- `application/x-www-form-urlencoded`
- `multipart/form-data`
- `text/plain`

→ **`application/json` 은 X** — JSON API 거의 항상 preflight.

---

## 3. 의미 — 옛 HTML 폼과 동등

```html
<!-- 옛 HTML 폼이 만들 수 있는 요청 -->
<form action="https://other.com/submit" method="POST" enctype="multipart/form-data">
  <input name="x">
</form>

<!-- 이런 폼은 어떻게든 만들 수 있음 — 새 보안 X -->
```

→ Simple request = "HTML 폼이 만들 수 있는 모든 요청" + 옵션.

---

## 4. Simple Request 예

### 예 1 — GET

```http
GET /api/users HTTP/1.1
Host: api.example.com
Origin: https://app.example.com
Accept: application/json

→ Simple ✅
```

### 예 2 — POST + form

```http
POST /api/users HTTP/1.1
Host: api.example.com
Origin: https://app.example.com
Content-Type: application/x-www-form-urlencoded

name=Alice&email=alice@example.com

→ Simple ✅
```

### 예 3 — POST + JSON (NOT simple)

```http
POST /api/users HTTP/1.1
Host: api.example.com
Origin: https://app.example.com
Content-Type: application/json   ← Not simple

{...}

→ Preflight 필요 ❌
```

### 예 4 — Custom 헤더 (NOT simple)

```http
GET /api/me HTTP/1.1
Host: api.example.com
Origin: https://app.example.com
Authorization: Bearer ...    ← Not simple

→ Preflight 필요 ❌
```

---

## 5. Simple Request 흐름

```
1. 브라우저: 직접 요청 (preflight X)
2. 서버: 응답 + CORS 헤더
3. 브라우저: Access-Control-Allow-Origin 확인
   - OK → JS 에 응답 전달
   - X  → JS 에 오류 (CORS error)
```

### 중요
- **요청 자체는 서버에 도달** — 서버가 처리 (DB 쓰기 등)
- 브라우저는 **응답 read** 만 차단
- → CSRF 위험 (`SameSite cookie` 가 별도 방어)

---

## 6. CORS 의 응답 검증 (Simple Request 도)

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://app.example.com
Content-Type: application/json
{...}
```

### 브라우저 검증

```
응답에 Access-Control-Allow-Origin 있음?
├ Yes:
│   ├ 값이 요청 Origin 과 일치? → JS 에 전달
│   ├ 값이 `*` → JS 에 전달 (credentials X 시)
│   └ 다른 값 → CORS error
└ No → CORS error (JS 가 응답 read X)
```

---

## 7. Credentials 모드

```javascript
// Simple request 도 credentials 옵션
fetch('https://other.com/api', {credentials: 'include'})
```

→ Cookie / Authorization 첨부. 추가 조건 (자세히 → [[cors-with-credentials]]).

---

## 8. fetch / XHR 의 자동 헤더

브라우저가 자동 추가:
- `Origin`
- `Referer`
- `User-Agent`
- `Sec-Fetch-*` (모던)

이 자동 헤더들은 Simple Request 판정에 영향 X.

---

## 9. Simple Request 의 흔한 케이스

### 9.1 GET (조회)
```javascript
fetch('https://api.com/users')
```

### 9.2 POST + form (전통 웹)
```javascript
const fd = new FormData();
fd.append('name', 'Alice');
fetch('https://api.com/submit', {method: 'POST', body: fd});
```

### 9.3 Image / Script 등
```html
<img src="https://other.com/photo.jpg">  ← 항상 simple
```

---

## 10. Preflight 가 필요한 경우 (NOT simple)

### 10.1 메서드
- PUT / DELETE / PATCH

### 10.2 Content-Type
- `application/json` (가장 흔함)
- `application/xml`
- `application/grpc`

### 10.3 Custom 헤더
- Authorization (Bearer / Basic)
- X-API-Key
- X-CSRF-Token

### 10.4 ReadableStream body

자세히 → [[preflight]]

---

## 11. JSON API 의 함정

```javascript
// 옛 — Simple 가능
fetch('/api/users', {
  method: 'POST',
  headers: {'Content-Type': 'application/x-www-form-urlencoded'},
  body: 'name=Alice&email=...'
});

// 모던 — Preflight 필요
fetch('/api/users', {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({name: 'Alice'})
});
```

→ 모던 JSON API 거의 항상 preflight. 정상.

---

## 12. 함정

### 함정 1 — Simple 이라고 안전 X
- 서버에 요청은 도달
- CSRF 위험 — SameSite Cookie + CSRF Token

### 함정 2 — 응답 read 만 차단
- 데이터 변경 (POST simple) 은 막지 X
- 서버가 항상 인증 / CSRF 검증

### 함정 3 — Image / Script 의 cross-origin
- 표시 / 실행 OK
- Pixel data / Script content read X (canvas tainted)

### 함정 4 — Credentials + Simple
- credentials: include 면 추가 조건
- Allow-Origin: * 와 결합 X

### 함정 5 — 일부 옛 브라우저
- 옛 IE 등은 simple 판정 X — preflight 다 보냄

### 함정 6 — Content-Type 자동 설정
- fetch 에 객체 body — Content-Type 자동 (application/json 으로)
- FormData 면 multipart 자동
- → 자동 설정 인지 필요

---

## 13. 학습 자료

- **Fetch standard** — https://fetch.spec.whatwg.org/#cors-protocol
- MDN CORS — Simple requests
- "CORS Tutorial" — web.dev

---

## 14. 관련

- [[cors]] — CORS hub
- [[preflight]] — 복잡 요청
- [[cors-with-credentials]] — credentials 모드
- [[../methods/post]] — POST + Content-Type
- [[../headers/entity-headers]] — Content-Type
