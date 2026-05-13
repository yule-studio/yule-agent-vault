---
title: "Last-Modified & If-Modified-Since"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T22:22:00+09:00
tags:
  - network
  - http
  - caching
  - last-modified
---

# Last-Modified & If-Modified-Since

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 시간 기반 캐시 검증 |

**[[caching|↑ Caching]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

자원의 **수정 시간** 으로 캐시 검증. ETag 보다 옛 + 정밀도 낮음 (초 단위) — 모던은
보통 둘 다 함께.

---

## 2. Last-Modified 헤더

```http
HTTP/1.1 200 OK
Last-Modified: Tue, 13 May 2026 10:00:00 GMT
```

### 형식
- HTTP date (RFC 5322 / 9110) — GMT 강제
- `Day, DD Mon YYYY HH:MM:SS GMT`

---

## 3. If-Modified-Since (GET)

```http
GET /api/users/123 HTTP/1.1
If-Modified-Since: Tue, 13 May 2026 10:00:00 GMT

응답:
  HTTP/1.1 304 Not Modified
  (또는)
  HTTP/1.1 200 OK
  Last-Modified: Tue, 13 May 2026 11:30:00 GMT
  {...new body...}
```

### 의미
- 클라가 받은 마지막 시간 이후 변경됐는지
- 변경 X → 304
- 변경 O → 200 + 새 body

---

## 4. If-Unmodified-Since (PUT / DELETE)

```http
PUT /api/users/123 HTTP/1.1
If-Unmodified-Since: Tue, 13 May 2026 10:00:00 GMT
{...}

→ 200 OK   (변경 안 됐으면)
또는
→ 412 Precondition Failed (그 시간 후 변경)
```

→ 낙관적 잠금의 시간 버전. ETag 의 If-Match 와 비슷.

---

## 5. Last-Modified 의 한계

### 5.1 초 단위 정밀도

```
12:00:00.500 에 첫 수정 → Last-Modified: ...12:00:00...
12:00:00.999 에 두 번째 수정 → Last-Modified: ...12:00:00... (같음)
```

→ 같은 초 안의 여러 수정 — 감지 X.

### 5.2 시계 동기 의존
- 서버 / 클라 시간이 다르면 오작동
- NTP 필수

### 5.3 시간이 의미 없는 자원
- DB 의 동적 자원 — "수정 시간" 정의 모호
- ETag (해시) 가 명확

---

## 6. ETag 와 함께 사용

```http
HTTP/1.1 200 OK
ETag: "v1.0-abc"
Last-Modified: Tue, 13 May 2026 10:00:00 GMT
```

```http
[조건부 요청]
GET /api/users/123 HTTP/1.1
If-None-Match: "v1.0-abc"
If-Modified-Since: Tue, 13 May 2026 10:00:00 GMT
```

### 평가 순서 (RFC 9111)
1. **If-Match / If-None-Match** (ETag) — 우선
2. **If-Modified-Since / If-Unmodified-Since** — fallback

→ ETag 있으면 그것 사용, 없으면 시간.

---

## 7. 정적 파일 — 자동 처리

### Nginx
```nginx
location /static/ {
    expires 1y;
    add_header Cache-Control "public, immutable";
    # Last-Modified 자동 — 파일 mtime
    # ETag 자동
}
```

### Apache
```apache
ExpiresActive On
ExpiresDefault "access plus 1 year"
# Last-Modified, ETag 자동
```

### Express (Node.js)
```javascript
app.use(express.static('public', {
    maxAge: '1y',
    etag: true,
    lastModified: true
}));
```

---

## 8. 동적 자원 — 응용 코드

### Express
```javascript
app.get('/api/users/:id', (req, res) => {
    const user = getUser(req.params.id);
    const lastMod = user.updated_at.toUTCString();
    
    if (req.headers['if-modified-since']) {
        if (new Date(req.headers['if-modified-since']) >= user.updated_at) {
            return res.status(304).end();
        }
    }
    
    res.set('Last-Modified', lastMod);
    res.json(user);
});
```

### FastAPI
```python
from fastapi import Response, Request
from email.utils import format_datetime, parsedate_to_datetime

@app.get("/api/users/{user_id}")
def get_user(user_id: int, request: Request, response: Response):
    user = get_from_db(user_id)
    
    ims = request.headers.get("if-modified-since")
    if ims:
        if parsedate_to_datetime(ims) >= user.updated_at:
            return Response(status_code=304)
    
    response.headers["Last-Modified"] = format_datetime(user.updated_at)
    return user
```

---

## 9. 함정

### 함정 1 — 초 단위 정밀도
빠른 변경 (배치 / 대량 import) 시 동일 초 — 감지 X. ETag 권장.

### 함정 2 — 시계 동기 미동기화
NTP 안 됨 → 캐시 오작동.

### 함정 3 — Timezone 무시
**GMT 강제**. Local time 사용 시 RFC 위반.

### 함정 4 — `updated_at` 의 의미
DB 의 trigger / ORM 이 자동 갱신 — 실제 자원 변경 안 됐는데 시간 변함. ETag (content hash) 가 정확.

### 함정 5 — 파일 시스템 mtime
git pull / rsync 후 mtime 갱신 — 실제 내용 같지만 캐시 invalid. `-T` 옵션 (rsync) 으로 보존.

### 함정 6 — Last-Modified 헤더 없으면
Heuristic caching (Cache-Control 도 없으면) → 의도 X 캐시.

### 함정 7 — If-Modified-Since 의 `0` / 옛 시간
"매번 받아라" — Cache-Control: no-cache 가 명확.

---

## 10. 학습 자료

- **RFC 9110** Section 8.8.2 (Last-Modified), 13 (Conditional)
- **RFC 9111** Section 4.3
- MDN Last-Modified / If-Modified-Since

---

## 11. 관련

- [[caching]] — Caching hub
- [[etag-conditional]] — 비교
- [[cache-control]] — 결합
- [[../headers/response-headers]] — Last-Modified 위치
- [[../status-codes/3xx-redirection]] — 304
