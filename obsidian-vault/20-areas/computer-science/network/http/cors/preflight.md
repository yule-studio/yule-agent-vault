---
title: "CORS Preflight (OPTIONS)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T23:20:00+09:00
tags:
  - network
  - http
  - cors
  - preflight
  - options
---

# CORS Preflight (OPTIONS)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | OPTIONS preflight 흐름 / 헤더 / 캐싱 |

**[[cors|↑ CORS]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

복잡한 (NOT simple) cross-origin 요청 전 브라우저가 **OPTIONS** 로 허락 확인.
실제 요청 전 추가 RTT.

---

## 2. Preflight 가 발생하는 조건

Simple Request 조건 위반 시 (자세히 → [[simple-request]]):
- 메서드: PUT / DELETE / PATCH / CONNECT / TRACE
- Content-Type: `application/json` / `application/xml` / ...
- Custom 헤더: `Authorization`, `X-API-Key`, `X-CSRF-Token` 등
- ReadableStream body

---

## 3. 전체 흐름

```
1. 브라우저: 복잡 요청 감지
   ↓
2. 브라우저: OPTIONS preflight 송신
   ├ Origin
   ├ Access-Control-Request-Method
   └ Access-Control-Request-Headers
   ↓
3. 서버: OPTIONS 응답
   ├ Access-Control-Allow-Origin
   ├ Access-Control-Allow-Methods
   ├ Access-Control-Allow-Headers
   ├ Access-Control-Allow-Credentials (옵션)
   └ Access-Control-Max-Age
   ↓
4. 브라우저: 응답 검증
   ├ Origin 매칭? ✓
   ├ Method 허용? ✓
   ├ Headers 허용? ✓
   └ OK → 실제 요청
   ↓
5. 브라우저: 실제 요청 (e.g., PUT)
   ↓
6. 서버: 실제 응답 (CORS 헤더 포함)
   ↓
7. 브라우저: JS 에 응답 전달
```

---

## 4. Preflight 요청 예

```http
OPTIONS /api/users/123 HTTP/1.1
Host: api.example.com
Origin: https://app.example.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: Content-Type, Authorization
```

### 자동 추가 (브라우저)
- Origin
- Access-Control-Request-Method (실제 요청의 메서드)
- Access-Control-Request-Headers (실제 요청의 custom 헤더)
- 일반 헤더 (User-Agent, Accept 등)

---

## 5. Preflight 응답 예

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization, X-Request-ID
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 86400
Vary: Origin
```

### 응답 코드
- **204 No Content** — 권장
- **200 OK** — OK

---

## 6. Access-Control-Allow-Methods

```http
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Methods: *           (모든 — credentials X 시)
```

- 응용이 명시
- Simple methods (GET/HEAD/POST) 는 명시 안 해도 OK

---

## 7. Access-Control-Allow-Headers

```http
Access-Control-Allow-Headers: Content-Type, Authorization, X-Request-ID
Access-Control-Allow-Headers: *           (모든 — credentials X 시)
```

- 요청의 Custom 헤더 명시
- `Authorization` 은 명시 필수 (wildcard `*` 에 포함 X)
- 자주 — `Content-Type, Authorization, X-Requested-With`

### Wildcard 의 함정

```http
Access-Control-Allow-Headers: *
```

- 모든 헤더 허용 (credentials X 시만)
- 그러나 `Authorization` 은 별도 명시 필요 (특별 처리)

---

## 8. Access-Control-Max-Age

```http
Access-Control-Max-Age: 86400        (24 시간)
```

### 의미
- 브라우저가 preflight 결과 캐시
- 같은 (method, headers) 의 다음 요청 — preflight skip

### 브라우저별 한계

| 브라우저 | 한계 |
| --- | --- |
| Chrome | 7200 (2 시간) — 이후 잘림 |
| Firefox | 86400 (24 시간) |
| Safari | 600 (10 분) |

→ 작은 한계로 잘림. 자주 preflight 발생.

### 권장
```http
Access-Control-Max-Age: 86400
```

- 큰 값 OK (브라우저가 적당히 자름)
- 디버깅 시 짧게 (Chrome DevTools `Disable Cache`)

---

## 9. Preflight 의 비용

### 추가 RTT
```
Without preflight:
  Request 1 RTT
  Response

With preflight:
  Preflight 1 RTT
  Preflight response
  Request 1 RTT
  Response
  → 2 RTT
```

### 영향
- 첫 요청 latency 2 배
- 캐시 (Max-Age) 후 동일

### 최적화
- 큰 Max-Age (86400)
- API 일관 (메서드 / Content-Type)
- Single page app: 첫 요청 후 같음

---

## 10. Preflight 와 인증

### 함정 — OPTIONS 에 인증 요구

```python
@app.before_request
def require_auth():
    if not request.headers.get('Authorization'):
        return 401

# 문제: OPTIONS 도 401 → preflight 실패 → 본 요청 X
```

### 해결 — OPTIONS 면제

```python
@app.before_request
def require_auth():
    if request.method == 'OPTIONS':
        return None    # OPTIONS 면제
    if not request.headers.get('Authorization'):
        return 401
```

→ OPTIONS 는 인증 X — 어차피 Authorization 헤더 안 보냄 (브라우저 자동).

---

## 11. 서버 구현 — 일반 패턴

### Nginx

```nginx
location /api/ {
    # CORS Preflight
    if ($request_method = 'OPTIONS') {
        add_header 'Access-Control-Allow-Origin' 'https://app.example.com' always;
        add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;
        add_header 'Access-Control-Allow-Headers' 'Content-Type, Authorization' always;
        add_header 'Access-Control-Allow-Credentials' 'true' always;
        add_header 'Access-Control-Max-Age' 86400 always;
        add_header 'Content-Length' 0;
        add_header 'Content-Type' 'text/plain charset=UTF-8';
        return 204;
    }
    
    # 실제 요청
    add_header 'Access-Control-Allow-Origin' 'https://app.example.com' always;
    add_header 'Access-Control-Allow-Credentials' 'true' always;
    proxy_pass http://backend;
}
```

### Express (Node.js)

```javascript
const cors = require('cors');
app.use(cors({
    origin: 'https://app.example.com',
    methods: ['GET', 'POST', 'PUT', 'DELETE'],
    allowedHeaders: ['Content-Type', 'Authorization'],
    credentials: true,
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
    allow_credentials=True,
    max_age=86400,
)
```

### Spring Boot

```java
@Configuration
public class CorsConfig {
    @Bean
    public WebMvcConfigurer corsConfigurer() {
        return new WebMvcConfigurer() {
            @Override
            public void addCorsMappings(CorsRegistry registry) {
                registry.addMapping("/api/**")
                    .allowedOrigins("https://app.example.com")
                    .allowedMethods("GET", "POST", "PUT", "DELETE")
                    .allowedHeaders("Content-Type", "Authorization")
                    .allowCredentials(true)
                    .maxAge(86400);
            }
        };
    }
}
```

---

## 12. 함정

### 함정 1 — OPTIONS 에 인증 요구
preflight 실패 → 본 요청 X. **OPTIONS 면제**.

### 함정 2 — Max-Age 너무 짧음
- 기본 0 → 매 요청 preflight
- 86400 권장

### 함정 3 — Wildcard + Credentials
- `Allow-Origin: *` + `Allow-Credentials: true` — 브라우저 거부

### 함정 4 — Allow-Headers 의 누락
- 요청 의 모든 custom 헤더 명시 (Authorization, Content-Type, X-CSRF-Token 등)

### 함정 5 — Vary: Origin 누락
- 여러 origin 지원 시 → 캐시가 다른 origin 의 응답 잘못 반환

### 함정 6 — Method override
- `X-HTTP-Method-Override: PUT` 으로 POST 위장 — CORS 우회 시도. 차단.

### 함정 7 — 브라우저 별 Max-Age 한계
Safari 10 분 — 자주 preflight. 캐시 최적화 한계.

### 함정 8 — Preflight 의 부하
큰 사이트 — preflight 만 수만 RPS. CDN 캐시 / Edge 처리.

---

## 13. 디버깅

```bash
# curl 로 preflight 시뮬레이션
curl -X OPTIONS https://api.example.com/users/123 \
  -H "Origin: https://app.example.com" \
  -H "Access-Control-Request-Method: PUT" \
  -H "Access-Control-Request-Headers: Content-Type, Authorization" \
  -i
```

### Chrome DevTools
- Network 탭에서 OPTIONS 요청 확인
- Failed CORS 시 콘솔에 상세 메시지

---

## 14. 학습 자료

- **Fetch standard** — Preflight 절
- MDN CORS preflight
- "CORS Preflight Caching" — Cloudflare

---

## 15. 관련

- [[cors]] — CORS hub
- [[simple-request]] — 면제 조건
- [[cors-with-credentials]] — credentials 모드
- [[../methods/options]] — OPTIONS 메서드
- [[../headers/response-headers]] — Access-Control-*
