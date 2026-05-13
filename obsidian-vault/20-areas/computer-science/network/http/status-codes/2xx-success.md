---
title: "2xx — Success"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T21:16:00+09:00
tags:
  - network
  - http
  - status-codes
  - 2xx
---

# 2xx — Success

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 200/201/202/203/204/205/206/207/208/226 |

**[[status-codes|↑ Status Codes]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

"요청 처리 성공". body 는 자원의 표현 또는 처리 결과.

---

## 2. 표준 코드

### 200 OK

```http
GET /users/123 HTTP/1.1

HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 73

{"id":123,"name":"Alice","email":"alice@example.com"}
```

가장 흔한 성공:
- GET — 자원 표현
- POST — 처리 결과
- PUT — 갱신된 자원

### 201 Created

```http
POST /users HTTP/1.1
Content-Type: application/json
{"name":"Alice"}

HTTP/1.1 201 Created
Location: /users/123
Content-Type: application/json
{"id":123,"name":"Alice"}
```

#### 사용
- POST 로 새 자원 생성
- PUT 으로 새 자원 생성 (URI 클라 지정)

#### Location 헤더 권장
- 새 자원의 URI
- 클라가 후속 조회 가능

### 202 Accepted

```http
POST /api/jobs/large HTTP/1.1
{...}

HTTP/1.1 202 Accepted
Location: /api/jobs/abc/status
Content-Type: application/json
{"job_id":"abc","status":"pending"}
```

#### 사용
- 비동기 처리 시작 — 아직 완료 X
- Long-running job (배치, 영상 변환, 발송)
- 클라가 polling 또는 webhook 으로 결과 확인

#### 폴링 패턴
```
GET /api/jobs/abc/status
→ 202 Accepted (계속)
→ 200 OK (완료) + 결과
```

### 203 Non-Authoritative Information

```http
HTTP/1.1 203 Non-Authoritative Information
```

Proxy / Transformer 가 본 응답을 변경했을 때:
- 옛 trans-coding proxy
- 거의 사용 X — 200 으로 대체

### 204 No Content

```http
DELETE /users/123 HTTP/1.1

HTTP/1.1 204 No Content
```

#### 사용
- 성공 + body X
- DELETE, PUT (갱신 후 body 안 보냄), POST 의 일부

#### 헤더
- `Content-Length` 없음 (또는 0)
- 페이지 새로고침 X (브라우저)

#### 흔한 패턴
```
204 No Content (성공)
vs
404 Not Found (자원 없음)
vs
410 Gone (영구 삭제)
```

### 205 Reset Content

```http
HTTP/1.1 205 Reset Content
```

폼 제출 후 폼 초기화 요청:
- 클라가 입력 UI 리셋
- 실용 거의 없음 — 204 + JS 가 대체

### 206 Partial Content

```http
GET /video.mp4 HTTP/1.1
Range: bytes=1000-1999

HTTP/1.1 206 Partial Content
Content-Range: bytes 1000-1999/50000
Content-Length: 1000
Content-Type: video/mp4

[1000 bytes]
```

#### 사용
- Range Request 응답
- 다운로드 재개, 영상 seeking, 병렬 다운로드

자세히 → [[../performance/range-requests]]

### 207 Multi-Status (WebDAV)

```http
HTTP/1.1 207 Multi-Status
Content-Type: application/xml

<multistatus>
  <response>
    <href>/file1</href>
    <status>HTTP/1.1 200 OK</status>
  </response>
  <response>
    <href>/file2</href>
    <status>HTTP/1.1 403 Forbidden</status>
  </response>
</multistatus>
```

여러 자원에 대한 결과 — WebDAV. 일반 HTTP 거의 X.

### 208 Already Reported (WebDAV)
207 안의 중복 자원 표시.

### 226 IM Used (RFC 3229)
Instance Manipulation — delta encoding. 거의 X.

---

## 3. 코드 선택 가이드

### POST 응답

| 시나리오 | 코드 |
| --- | --- |
| 자원 생성 + body | **201 Created** + Location |
| 자원 생성, body X | 201 Created |
| 처리 + body | 200 OK |
| 처리, body X | 204 No Content |
| 비동기 시작 | **202 Accepted** + Location |

### PUT 응답

| 시나리오 | 코드 |
| --- | --- |
| 갱신 + body | 200 OK |
| 갱신, body X | 204 No Content |
| 새로 생성 | 201 Created |

### DELETE 응답

| 시나리오 | 코드 |
| --- | --- |
| 즉시 삭제 + body | 200 OK |
| 즉시 삭제, body X | **204 No Content** |
| 비동기 삭제 | 202 Accepted |
| 이미 삭제됨 | 204 또는 404 |

### GET 응답

| 시나리오 | 코드 |
| --- | --- |
| 자원 반환 | **200 OK** |
| Range 응답 | 206 Partial Content |
| 빈 자원 | 204 No Content (드뭎) |

---

## 4. 헤더와 결합

### 201 Created + Location
```http
HTTP/1.1 201 Created
Location: /api/users/123
```

### 202 Accepted + 폴링
```http
HTTP/1.1 202 Accepted
Location: /api/jobs/abc/status
Retry-After: 5
```

### 200 OK + Cache
```http
HTTP/1.1 200 OK
Cache-Control: max-age=300
ETag: "v1"
Content-Type: application/json
```

### 206 Partial Content + Range
```http
HTTP/1.1 206 Partial Content
Content-Range: bytes 1000-1999/50000
Accept-Ranges: bytes
```

---

## 5. 응용 / 프레임워크

### Express (Node.js)
```javascript
res.status(201).location('/users/123').json({id: 123});
res.status(204).end();
res.status(202).json({job_id: 'abc'});
```

### FastAPI
```python
from fastapi import status, Response
@app.post("/users", status_code=status.HTTP_201_CREATED)
def create_user(...):
    return {...}

@app.delete("/users/{id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_user(id: int):
    return Response(status_code=204)
```

### Spring (Java)
```java
@PostMapping("/users")
public ResponseEntity<User> create(@RequestBody User u) {
    User saved = service.save(u);
    return ResponseEntity
        .created(URI.create("/users/" + saved.getId()))
        .body(saved);
}
```

---

## 6. 함정

### 함정 1 — 모든 성공을 200
RESTful 응용은 201 / 202 / 204 를 활용 — 의미 전달.

### 함정 2 — 201 의 Location 누락
클라가 후속 URI 모름.

### 함정 3 — 204 의 body
body 있으면 RFC 위반. 클라가 무시.

### 함정 4 — 200 OK + JSON error body
```
HTTP/1.1 200 OK
{"error": "Validation failed"}    ← 안티패턴
```

→ 4xx 사용. HTTP 의미 활용.

### 함정 5 — 202 의 후속 폴링 미정의
Location + Retry-After 권장. 클라가 알기 쉽게.

### 함정 6 — 206 의 단일 Range 만
multiple range (`Range: bytes=0-99,200-299`) 응답은 `multipart/byteranges` — 복잡.

---

## 7. 학습 자료

- **RFC 9110** Section 15.3 (2xx)
- MDN 2xx success status

---

## 8. 관련

- [[status-codes]] — hub
- [[1xx-informational]] — 100, 103
- [[3xx-redirection]] — 304
- [[../methods/methods]] — 메서드별 응답
- [[../headers/response-headers]] — Location, Retry-After
- [[../performance/range-requests]] — 206
