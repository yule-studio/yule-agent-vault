---
title: "GET — 조회"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:38:00+09:00
tags:
  - network
  - http
  - get
---

# GET — 조회

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 동작 / Query string / 한계 / 함정 |

**[[methods|↑ Methods]]** · **[[../http|↑↑ HTTP]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **안전** | ✅ Safe |
| **멱등** | ✅ Idempotent |
| **캐시 가능** | ✅ |
| **Body** | ❌ (의미 있는 사용 없음) |
| **도입** | HTTP/0.9 |

---

## 1. 한 줄 정의

**자원의 표현 (representation) 을 조회**. HTTP 의 기본 — 모든 GET 은 서버 상태를
변경하지 X. 응답은 캐시 가능.

---

## 2. 요청 / 응답

```http
GET /api/users/123 HTTP/1.1
Host: api.example.com
Accept: application/json

```

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 73
Cache-Control: max-age=300
ETag: "v1.0-abc123"

{"id":123,"name":"Alice","email":"alice@example.com"}
```

---

## 3. Query String — Parameters

```
GET /search?q=hello&page=2&limit=10 HTTP/1.1
```

### URL Encoding (Percent Encoding, RFC 3986)

```
공백 → %20 또는 +
한글 "안녕" → %EC%95%88%EB%85%95 (UTF-8 + percent)
특수문자 ?&=#+ → 각각 %3F %26 %3D %23 %2B
```

### Python 예
```python
from urllib.parse import urlencode
params = {"q": "안녕", "page": 2}
url = "https://api.com/search?" + urlencode(params)
# → "?q=%EC%95%88%EB%85%95&page=2"
```

---

## 4. GET 의 한계

### 4.1 URL 길이
- HTTP 표준은 한계 X
- **실제 한계** — 브라우저 / 서버 / proxy 마다:
  - Chrome: 2 MB+
  - IE: 2083
  - Nginx default: 8 KB
  - Apache default: 8 KB

→ 큰 데이터는 POST.

### 4.2 Body 사용 X
- 표준은 GET body 금지 X (RFC 9110 "의미 정의 X")
- **하지만**:
  - 캐시가 body 무시
  - Proxy 가 body 폐기
  - 일부 라이브러리 X
- **실용**: GET body 사용 X. Elasticsearch 의 GET _search 는 예외 (POST 권장).

### 4.3 Body 없음 → 데이터 보내려면 Query String
- URL 노출 → 로그에 남음
- 민감 정보 (비밀번호, 토큰) 보내지 X
- **Authorization 헤더** 사용

---

## 5. 조건부 GET — 캐시

### 5.1 ETag 기반
```http
GET /api/users/123 HTTP/1.1
If-None-Match: "v1.0-abc123"

HTTP/1.1 304 Not Modified
ETag: "v1.0-abc123"
(body 없음)
```

### 5.2 시간 기반
```http
GET /api/users/123 HTTP/1.1
If-Modified-Since: Mon, 13 May 2026 10:00:00 GMT

HTTP/1.1 304 Not Modified
(또는 200 OK + 새 body)
```

자세히 → [[../caching/etag-conditional]], [[../caching/last-modified]]

---

## 6. Range GET — 부분 조회 (RFC 7233)

```http
GET /video.mp4 HTTP/1.1
Range: bytes=0-1023

HTTP/1.1 206 Partial Content
Content-Range: bytes 0-1023/4096
Content-Length: 1024

(첫 1024 byte)
```

- 동영상 seeking
- 다운로드 재개
- 병렬 다운로드 (axel, aria2)

자세히 → [[../performance/range-requests]]

---

## 7. 응답 코드

| 코드 | 의미 |
| --- | --- |
| **200 OK** | 정상 응답 |
| **204 No Content** | 빈 응답 (body X) |
| **206 Partial Content** | Range request 응답 |
| **301 / 302 / 307 / 308** | Redirect |
| **304 Not Modified** | 캐시 사용 |
| **404 Not Found** | 자원 없음 |
| **410 Gone** | 영구 삭제 |

---

## 8. curl 예

```bash
# 기본
curl https://api.example.com/users/123

# 헤더 추가
curl -H "Accept: application/json" \
     -H "Authorization: Bearer TOKEN" \
     https://api.example.com/users/123

# Query string
curl -G --data-urlencode "q=한글 검색" --data "page=2" \
     https://api.example.com/search

# 응답 헤더 + body
curl -i https://example.com/

# 헤더만
curl -I https://example.com/    # 실제로 HEAD

# Verbose
curl -v https://example.com/
```

---

## 9. fetch (JS)

```javascript
// 기본
const r = await fetch("/api/users/123");
const data = await r.json();

// 헤더
const r = await fetch("/api/users/123", {
  headers: {
    "Accept": "application/json",
    "Authorization": "Bearer TOKEN"
  }
});

// Query string
const params = new URLSearchParams({q: "hello", page: 2});
const r = await fetch(`/api/search?${params}`);
```

---

## 10. 함정

### 함정 1 — GET 으로 상태 변경
RFC 위반. 검색 봇 / 프리페치가 호출 → 의도 X 영향.

#### 예시 (잘못된 코드)
```
GET /api/users/123/delete   ← 절대 X
```

→ DELETE 메서드 사용.

### 함정 2 — Query string 의 비밀
URL 이 로그 / Referer / 브라우저 history 에 남음. 비밀번호 / 토큰 X.

### 함정 3 — URL 길이
서버 / Proxy 별 한계. 8 KB 권장 한계.

### 함정 4 — GET Body 사용
거의 안 됨 — 일부 라이브러리 / 캐시 무시.

### 함정 5 — 캐시 가능성 무시
민감 정보 GET → CDN / 브라우저 캐시 위험. `Cache-Control: no-store`.

### 함정 6 — Idempotent 가정 위반
GET 호출 시마다 카운터 증가 (analytics) — 사실상 비멱등. 분리 필요.

### 함정 7 — Pagination
큰 데이터 → `?page=1&limit=20` 필수.

### 함정 8 — Encoding 잘못
UTF-8 vs Latin-1 — 한글 깨짐. 항상 UTF-8.

---

## 11. 패턴 / 모범

### 11.1 Resource-oriented
```
GET /users/123           ← 좋음 (자원 명사)
GET /getUser?id=123      ← 안 좋음 (동사)
```

### 11.2 Filter / Sort / Paginate
```
GET /users?role=admin&sort=created_at&page=1&limit=20
```

### 11.3 Embedded resources
```
GET /users/123?include=orders,addresses
```

### 11.4 Sparse Fieldsets
```
GET /users/123?fields=name,email
```

JSON:API 표준.

---

## 12. 학습 자료

- **RFC 9110** Section 9.3.1
- **RFC 3986** — URI / URL Encoding
- MDN GET — https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/GET
- "HTTP idempotency" 시리즈

---

## 13. 관련

- [[methods]] — Methods hub
- [[post]] — 대비
- [[head]] — GET 의 헤더만 버전
- [[../caching/caching]] — GET 의 캐시
- [[../rest/rest]] — REST 에서의 GET
