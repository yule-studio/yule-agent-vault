---
title: "PUT — 전체 갱신 / 생성"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:46:00+09:00
tags:
  - network
  - http
  - put
---

# PUT — 전체 갱신 / 생성

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 멱등 / 전체 교체 / Conditional PUT |

**[[methods|↑ Methods]]** · **[[../http|↑↑ HTTP]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **안전** | ❌ |
| **멱등** | ✅ |
| **캐시 가능** | ❌ |
| **Body** | ✅ |
| **도입** | HTTP/1.1 |

---

## 1. 한 줄 정의

특정 URI 의 **자원 전체를 본문으로 교체** (또는 없으면 생성). PUT 의 핵심은
"전체" — PATCH 와 대비.

---

## 2. 요청 / 응답

```http
PUT /api/users/123 HTTP/1.1
Host: api.example.com
Content-Type: application/json
Content-Length: 73

{"id":123,"name":"Alice","email":"alice@new.com","age":31}
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"id":123,"name":"Alice","email":"alice@new.com","age":31}
```

또는:

```http
HTTP/1.1 204 No Content
```

---

## 3. PUT vs POST

| 측면 | PUT | POST |
| --- | --- | --- |
| URI | **자원 URI 직접** (`/users/123`) | **컬렉션** (`/users`) |
| 의미 | 교체 / 생성 | 처리 / 추가 |
| 멱등 | ✅ | ❌ |
| 응답 | 200 / 204 | 201 (Created) |
| ID 결정 | **클라이언트** | **서버** |

### 예
```
POST /users + body → 서버가 ID 생성 → 201 Created + Location: /users/123
PUT /users/123 + body → 클라가 ID 지정 → 200 OK
```

---

## 4. PUT vs PATCH

| 측면 | PUT | PATCH |
| --- | --- | --- |
| 범위 | **전체** 교체 | **부분** 수정 |
| 멱등 | ✅ | ❌ (보장 X) |
| 본문 | 전체 자원 | 변경 사항 |
| 누락 필드 | 삭제 / 기본값 | 그대로 |

### 예
원본:
```json
{"id":123,"name":"Alice","age":30,"email":"alice@old.com"}
```

PUT (전체):
```http
PUT /users/123
{"id":123,"name":"Alice","age":31,"email":"alice@new.com"}

→ 모든 필드 교체. email 빠뜨리면 삭제 또는 default
```

PATCH (부분):
```http
PATCH /users/123
{"email":"alice@new.com"}

→ email 만 갱신, 나머지 유지
```

자세히 → [[patch]]

---

## 5. 멱등성 — 왜 ✅

```
PUT /users/123 {name: "Alice"}
PUT /users/123 {name: "Alice"}    ← 같은 결과
PUT /users/123 {name: "Alice"}    ← 같은 결과
```

→ 네트워크 실패 시 재시도 안전.

### 예외 — 응용 로직이 멱등 깨뜨림
```
PUT /users/123/timestamp     ← 매번 updated_at 갱신 → 사실상 비멱등
```

응용 설계 시 주의.

---

## 6. Conditional PUT — 동시성 제어

### 6.1 If-Match (낙관적 잠금)

```http
GET /users/123 → ETag: "v1"
PUT /users/123 + If-Match: "v1"  → 200 OK + ETag: "v2"
PUT /users/123 + If-Match: "v1"  → 412 Precondition Failed (이미 v2)
```

→ "마지막 본 버전" 일 때만 갱신.

### 6.2 If-Unmodified-Since
```http
PUT /users/123 + If-Unmodified-Since: <timestamp>
```

### 6.3 If-None-Match: *
"존재하지 않을 때만 생성":
```http
PUT /users/123 + If-None-Match: *
→ 201 Created (없었으면)
→ 412 Precondition Failed (있으면)
```

---

## 7. 응답 코드

| 코드 | 의미 |
| --- | --- |
| **200 OK** | 갱신 성공 + body |
| **201 Created** | 새로 생성됨 + Location |
| **204 No Content** | 갱신 성공, body X |
| **400 Bad Request** | 검증 실패 |
| **409 Conflict** | 충돌 |
| **412 Precondition Failed** | If-Match 실패 |
| **413 Payload Too Large** | 본문 큼 |
| **415 Unsupported Media Type** | Content-Type 미지원 |

---

## 8. curl 예

```bash
# JSON
curl -X PUT https://api.example.com/users/123 \
  -H "Content-Type: application/json" \
  -d '{"id":123,"name":"Alice","email":"alice@new.com"}'

# 조건부
curl -X PUT https://api.example.com/users/123 \
  -H "If-Match: \"v1\"" \
  -H "Content-Type: application/json" \
  -d '{...}'

# 파일 업로드 (WebDAV / S3 스타일)
curl -X PUT https://s3.example.com/bucket/file.jpg \
  --upload-file photo.jpg
```

---

## 9. PUT 의 활용

### 9.1 REST API
```
PUT /users/123 → 사용자 갱신
PUT /products/abc → 제품 갱신
```

### 9.2 파일 업로드 (Object Storage)
```
PUT /bucket/file.jpg → S3 / R2 의 파일 업로드
```

### 9.3 WebDAV
```
PUT /files/document.docx → 파일 저장
```

### 9.4 설정
```
PUT /api/config/feature_flag → 설정 교체
```

---

## 10. 함정

### 함정 1 — PUT 으로 부분 갱신
누락 필드 삭제됨 → 의도 X. PATCH 사용.

### 함정 2 — 멱등 가정의 응용 위반
`updated_at` 자동 갱신 시 멱등 X. RFC 의 멱등은 "효과" — DB 의 부수 효과는 별개.

### 함정 3 — Body 의 ID 와 URL ID 불일치
```
PUT /users/123 + body: {id: 456}   ← 모순 → 4xx 가 안전
```

### 함정 4 — 동시 PUT
두 클라가 동시에 → 마지막 wins. **Conditional PUT** 필수.

### 함정 5 — PUT 으로 생성
RFC 허용 — 서버가 ID 클라이언트 결정 OK. 보통 POST.

### 함정 6 — Content-Type 미지정
서버가 처리 못 함 → 415.

---

## 11. PUT vs POST 결정 트리

```
URI 가 자원 식별?
├ Yes (예: /users/123)
│   ├ 전체 교체? → PUT
│   └ 부분 갱신? → PATCH
└ No (예: /users 컬렉션)
    └ POST (서버가 ID 생성)
```

---

## 12. 학습 자료

- **RFC 9110** Section 9.3.4
- MDN PUT
- "PUT vs PATCH" — REST 가이드

---

## 13. 관련

- [[methods]] — Methods hub
- [[patch]] — 부분 갱신
- [[post]] — 비교
- [[idempotency-safety]]
- [[../caching/etag-conditional]] — If-Match
