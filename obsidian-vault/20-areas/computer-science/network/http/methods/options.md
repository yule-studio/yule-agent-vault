---
title: "OPTIONS — 허용 메서드 조회 / CORS Preflight"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:58:00+09:00
tags:
  - network
  - http
  - options
  - cors
---

# OPTIONS — 허용 메서드 조회 / CORS Preflight

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Allow / CORS preflight / 사용 사례 |

**[[methods|↑ Methods]]** · **[[../http|↑↑ HTTP]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **안전** | ✅ |
| **멱등** | ✅ |
| **캐시 가능** | ❌ |
| **Body** | ❌ |
| **도입** | HTTP/1.1 |

---

## 1. 한 줄 정의

**자원이 지원하는 메서드 조회**. 가장 큰 실용 — **CORS preflight 요청** (브라우저가
자동 송신).

---

## 2. 두 가지 사용

### 2.1 메서드 조회 (옛 의도)

```http
OPTIONS /api/users/123 HTTP/1.1
Host: api.example.com

```

```http
HTTP/1.1 204 No Content
Allow: GET, POST, PUT, DELETE, OPTIONS
```

→ "이 URI 가 지원하는 메서드".

### 2.2 CORS Preflight (실제 보편 사용)

```http
OPTIONS /api/users/123 HTTP/1.1
Origin: https://app.example.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: Content-Type, Authorization

```

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400
```

→ 브라우저가 **실제 요청 전** 자동 호출.

자세히 → [[../cors/preflight]]

---

## 3. Server-wide OPTIONS

```http
OPTIONS * HTTP/1.1
Host: example.com

```

→ 서버 전체의 capability. 거의 사용 X.

---

## 4. CORS Preflight 흐름

```
브라우저 (app.com) ─→ Cross-origin 요청 시도 (api.com)
   ↓
"Simple request" 아님?
   ↓ (예: PUT, custom header, JSON body)
브라우저: OPTIONS preflight 자동 송신
   ↓
서버 응답 OK?
   ↓ 예
브라우저: 실제 PUT 요청
   ↓
응답 받음
```

### "Simple request" 조건 (Preflight 면제)
- 메서드: GET, HEAD, POST
- 헤더: Accept, Accept-Language, Content-Language, Content-Type (한정)
- Content-Type: application/x-www-form-urlencoded, multipart/form-data, text/plain

자세히 → [[../cors/simple-request]]

---

## 5. Access-Control-Max-Age

```http
Access-Control-Max-Age: 86400    (24 시간)
```

→ 브라우저가 preflight 결과 캐시. 매 요청 X.

### 브라우저별 한계
- Chrome: 최대 7200 초 (2 시간)
- Firefox: 24 시간
- 더 큰 값은 한계로

---

## 6. OPTIONS 의 응답 헤더

### 공통 (메서드 조회)
- **Allow**: 허용 메서드 목록

### CORS Preflight
- **Access-Control-Allow-Origin**
- **Access-Control-Allow-Methods**
- **Access-Control-Allow-Headers**
- **Access-Control-Allow-Credentials**
- **Access-Control-Max-Age**
- **Access-Control-Expose-Headers** (실제 요청의 응답에)

---

## 7. 응답 코드

| 코드 | 의미 |
| --- | --- |
| **200 OK** | 응답 (body 있을 수도) |
| **204 No Content** | 응답 (body X) — CORS preflight |
| **403 Forbidden** | CORS 거부 |
| **404 Not Found** | 자원 없음 |
| **405 Method Not Allowed** | OPTIONS 자체 차단 (드뭎) |

---

## 8. curl 예

```bash
# 메서드 조회
curl -X OPTIONS https://api.example.com/users/123 -i

# CORS preflight 시뮬레이션
curl -X OPTIONS https://api.example.com/users/123 \
  -H "Origin: https://app.example.com" \
  -H "Access-Control-Request-Method: PUT" \
  -H "Access-Control-Request-Headers: content-type, authorization" \
  -i
```

---

## 9. 서버 구현 — Nginx

```nginx
location /api/ {
    if ($request_method = 'OPTIONS') {
        add_header 'Access-Control-Allow-Origin' 'https://app.example.com';
        add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE';
        add_header 'Access-Control-Allow-Headers' 'Content-Type, Authorization';
        add_header 'Access-Control-Max-Age' 86400;
        add_header 'Content-Length' 0;
        add_header 'Content-Type' 'text/plain charset=UTF-8';
        return 204;
    }
    proxy_pass http://backend;
}
```

### Express (Node.js)

```javascript
// cors 미들웨어
const cors = require('cors');
app.use(cors({
  origin: 'https://app.example.com',
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  maxAge: 86400,
}));
```

### FastAPI

```python
from fastapi.middleware.cors import CORSMiddleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.example.com"],
    allow_methods=["*"],
    allow_headers=["*"],
    max_age=86400,
)
```

---

## 10. 함정

### 함정 1 — OPTIONS 비활성
일부 보안 정책이 차단 → CORS 깨짐. 명시적 허용.

### 함정 2 — 인증 / 인가 적용
OPTIONS 에 인증 요구 → preflight 실패 → 본 요청 X. **OPTIONS 는 인증 면제**.

### 함정 3 — Max-Age 너무 짧음
매 요청 preflight → latency ↑. 86400 권장.

### 함정 4 — Wildcard + Credentials
```
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
```
→ 브라우저가 거부. 명시 origin.

자세히 → [[../cors/cors-with-credentials]]

### 함정 5 — `*` 의 의미
- `Allow: *` (헤더) — 모든 헤더
- `Origin: *` — 모든 출처
- 일부 보안 헤더는 wildcard 처리 안 함

### 함정 6 — Preflight 의 부하
큰 사이트에 큰 부하 — CDN 캐시 / Max-Age 늘리기.

### 함정 7 — Method override
`X-HTTP-Method-Override: PUT` 으로 POST 위장 — CORS 우회 시도. 서버가 거부.

---

## 11. 학습 자료

- **RFC 9110** Section 9.3.7
- **CORS** (Fetch standard) — https://fetch.spec.whatwg.org/
- MDN OPTIONS / CORS preflight

---

## 12. 관련

- [[methods]] — Methods hub
- [[../cors/cors]] — CORS hub
- [[../cors/preflight]] — Preflight 자세히
- [[../cors/simple-request]] — Simple request 조건
- [[trace-connect]]
