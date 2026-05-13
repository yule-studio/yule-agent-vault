---
title: "POST — 생성 / 처리"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:42:00+09:00
tags:
  - network
  - http
  - post
---

# POST — 생성 / 처리

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Body / Content-Type / multipart / 멱등 X |

**[[methods|↑ Methods]]** · **[[../http|↑↑ HTTP]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **안전** | ❌ |
| **멱등** | ❌ |
| **캐시 가능** | △ (`Cache-Control` 명시 시만) |
| **Body** | ✅ (필수) |
| **도입** | HTTP/1.0 |

---

## 1. 한 줄 정의

서버에 **데이터를 보내 처리** 시킴. 새 자원 생성 / 폼 제출 / 명령 실행 / 검색 등.
가장 유연한 메서드 — "정확한 의미 없음" 의 위험.

---

## 2. 요청 / 응답

```http
POST /api/users HTTP/1.1
Host: api.example.com
Content-Type: application/json
Content-Length: 56

{"name":"Alice","email":"alice@example.com"}
```

```http
HTTP/1.1 201 Created
Location: /api/users/123
Content-Type: application/json

{"id":123,"name":"Alice","email":"alice@example.com"}
```

→ `201 Created` + `Location` 헤더로 새 자원 URI.

---

## 3. Content-Type 별 본문

### 3.1 application/json (모던 API)

```http
Content-Type: application/json

{"name":"Alice","age":30}
```

### 3.2 application/x-www-form-urlencoded (HTML 폼 기본)

```http
Content-Type: application/x-www-form-urlencoded

name=Alice&age=30&email=alice%40example.com
```

GET 의 query string 과 같은 형식. 단순 폼.

### 3.3 multipart/form-data (파일 업로드)

```http
Content-Type: multipart/form-data; boundary=----X
Content-Length: ...

------X
Content-Disposition: form-data; name="username"

Alice
------X
Content-Disposition: form-data; name="avatar"; filename="me.jpg"
Content-Type: image/jpeg

<binary data>
------X--
```

자세히 → [[../headers/entity-headers]]

### 3.4 application/xml / text/xml
SOAP / XML-RPC.

### 3.5 application/octet-stream
임의 바이너리.

### 3.6 text/plain
단순 텍스트.

---

## 4. POST 의 4 가지 의미

POST 는 "서버가 처리" 만 정의 — 구체는 응용 결정:

### 4.1 자원 생성 (Create)
```http
POST /users
→ 201 Created + Location: /users/123
```

### 4.2 자원 처리 (Submit)
```http
POST /search          ← GET 으로 부족할 때 (큰 query)
POST /api/login
POST /api/logout
```

### 4.3 자원 추가 (Append)
```http
POST /users/123/comments
```

### 4.4 명령 실행 (RPC-like)
```http
POST /api/jobs/start
POST /api/users/123/activate
```

→ REST 의 자원 모델 외 — RPC 스타일.

---

## 5. 멱등성 — 왜 X

```
POST /orders (사용자 결제) 두 번 호출 →
두 주문 / 두 결제
```

→ 사용자가 새로고침 / 네트워크 재시도 시 위험.

### 5.1 Idempotency Key 패턴
```http
POST /api/orders HTTP/1.1
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000

{...}
```

서버:
- 같은 key 의 두 번째 요청 → 첫 응답 반환 (재처리 X)
- Stripe, Adyen, AWS API 가 사용

### 5.2 PRG 패턴 (Post-Redirect-Get)
폼 제출 후 새로고침 시 재제출 방지:
```
POST /submit → 302 Found → Location: /thanks
GET /thanks → 200 OK
```

---

## 6. 응답 코드

| 코드 | 의미 |
| --- | --- |
| **200 OK** | 처리 성공 (응답 body) |
| **201 Created** | 자원 생성 (Location 헤더) |
| **202 Accepted** | 비동기 처리 시작 |
| **204 No Content** | 처리 성공, body X |
| **400 Bad Request** | 검증 실패 |
| **401 Unauthorized** | 인증 |
| **403 Forbidden** | 인가 |
| **409 Conflict** | 충돌 (중복) |
| **413 Payload Too Large** | 본문 너무 큼 |
| **422 Unprocessable Entity** | 의미적 오류 (검증 실패) |
| **429 Too Many Requests** | rate limit |

---

## 7. POST 의 캐시

- 기본: **캐시 X**
- 명시 시 가능:
  ```http
  Cache-Control: public, max-age=300
  ```
- 일부 사용 — 비싼 검색 결과

### POST → GET → cache 패턴

```
POST /search → 결과 + 결과 ID
GET /search/results/{id} → 캐시된 결과
```

---

## 8. 100-continue 패턴

큰 본문 전송 전 서버 의사 확인:

```http
POST /upload HTTP/1.1
Content-Length: 1073741824    (1 GB)
Expect: 100-continue

(body 안 보내고 대기)

HTTP/1.1 100 Continue

(이제 body 보냄)

또는:

HTTP/1.1 417 Expectation Failed
(또는 다른 4xx)
```

큰 파일이 거부될 거면 전송 시작 전 안다.

---

## 9. curl 예

```bash
# JSON
curl -X POST https://api.example.com/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice","email":"alice@example.com"}'

# Form
curl -X POST https://example.com/submit \
  -d "name=Alice&email=alice@example.com"

# 파일 (multipart)
curl -X POST https://example.com/upload \
  -F "username=Alice" \
  -F "file=@photo.jpg"

# 파일 직접 (raw)
curl -X POST https://example.com/upload \
  --data-binary @photo.jpg \
  -H "Content-Type: image/jpeg"

# Idempotency-Key
curl -X POST https://api.com/orders \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '...'
```

---

## 10. fetch (JS)

```javascript
// JSON
const r = await fetch("/api/users", {
  method: "POST",
  headers: {"Content-Type": "application/json"},
  body: JSON.stringify({name: "Alice"})
});

// Form
const fd = new FormData();
fd.append("name", "Alice");
fd.append("file", fileInput.files[0]);
const r = await fetch("/upload", {method: "POST", body: fd});
// Content-Type 자동 multipart/form-data
```

---

## 11. 함정

### 함정 1 — 재시도로 중복 처리
네트워크 timeout → 클라가 재시도 → 두 번 처리. **Idempotency Key**.

### 함정 2 — 큰 본문
- HTTP/1.1 의 Content-Length 정확
- chunked 사용 시 끝 표시 (`0\r\n\r\n`)
- 서버의 `client_max_body_size` (Nginx) 확인

### 함정 3 — multipart 의 boundary
양쪽이 알아야 — Content-Type 헤더에 명시. 라이브러리가 자동.

### 함정 4 — POST 로 모든 것 (RPC-like)
REST 의 의미 잃음. GET (조회) / DELETE (삭제) 도 사용.

### 함정 5 — CSRF
POST 만으로 안전 X — SameSite 쿠키 / CSRF token 필수.

### 함정 6 — Content-Length / Transfer-Encoding 충돌
HTTP Request Smuggling 위험 — 둘 다 명시하지 X.

### 함정 7 — Body 인코딩
UTF-8 명시: `Content-Type: application/json; charset=UTF-8`.

---

## 12. 학습 자료

- **RFC 9110** Section 9.3.3
- MDN POST
- RFC 7578 (multipart/form-data)
- Stripe Idempotency 가이드

---

## 13. 관련

- [[methods]] — Methods hub
- [[put]], [[patch]] — 갱신 비교
- [[../headers/entity-headers]] — Content-Type 깊이
- [[../security/security]] — CSRF
- [[idempotency-safety]]
