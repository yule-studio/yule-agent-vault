---
title: "CORS (Hub)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T23:05:00+09:00
tags:
  - network
  - http
  - cors
---

# CORS (Hub) — Cross-Origin Resource Sharing

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | SOP + CORS 헤더 + Preflight + Credentials |

**[[../http|↑ HTTP]]** · **[[../../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

**다른 origin 의 자원 요청**을 브라우저가 허용 / 거부하는 메커니즘. Same-Origin
Policy (SOP) 의 보완. **응답 헤더** 가 결정.

---

## 2. Origin 의 정의

```
Origin = scheme + host + port

https://app.example.com:443  
http://app.example.com       (다름 — scheme 차이)
https://app.example.com:8080 (다름 — port 차이)
https://api.example.com      (다름 — host 차이)
```

→ 셋 중 하나만 달라도 cross-origin.

---

## 3. 왜 CORS 가 있는가

### 3.1 Same-Origin Policy (SOP)

브라우저의 기본 보안:
```
JS 가 다른 origin 의 응답 read X
```

이유 — CSRF / cross-site 데이터 도난 방어.

자세히 → [[same-origin-policy]]

### 3.2 정당한 cross-origin 필요

```
앱 도메인:  app.example.com (정적 호스팅)
API 도메인: api.example.com (백엔드)

→ 같은 회사지만 cross-origin
→ JS 가 fetch('https://api...') 못 함
```

→ CORS 가 정당한 cross-origin 허용.

---

## 4. CORS 동작

### 4.1 Simple Request

```http
GET /api/users HTTP/1.1
Host: api.example.com
Origin: https://app.example.com

HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://app.example.com
Content-Type: application/json
{...}
```

→ 응답에 `Access-Control-Allow-Origin` 있어야 브라우저가 JS 에 전달.

### 4.2 Preflight (복잡 요청)

복잡 요청 (PUT/DELETE/custom header/JSON body) 전:

```http
OPTIONS /api/users/123 HTTP/1.1
Origin: https://app.example.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: Content-Type, Authorization

HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400
```

→ Preflight 통과 후 실제 요청.

자세히 → [[preflight]], [[simple-request]]

---

## 5. CORS 헤더 전체

### 요청 (브라우저 자동)
- **Origin** — 출처
- **Access-Control-Request-Method** — preflight 만
- **Access-Control-Request-Headers** — preflight 만

### 응답 (서버가 설정)
- **Access-Control-Allow-Origin** — 허용 origin (또는 `*`)
- **Access-Control-Allow-Methods** — 허용 메서드
- **Access-Control-Allow-Headers** — 허용 헤더
- **Access-Control-Allow-Credentials** — Cookie / Authorization 허용
- **Access-Control-Max-Age** — preflight 캐시 시간
- **Access-Control-Expose-Headers** — JS 가 읽을 수 있는 응답 헤더

---

## 6. Allow-Origin 의 값

### 단일 origin (명시)
```http
Access-Control-Allow-Origin: https://app.example.com
```

### Wildcard
```http
Access-Control-Allow-Origin: *
```

⚠️ **Credentials 와 결합 X** — `*` + credentials 면 브라우저 거부.

### 동적 (응용 코드)
```python
origin = request.headers.get('Origin')
if origin in ALLOWED_ORIGINS:
    response.headers['Access-Control-Allow-Origin'] = origin
    response.headers['Vary'] = 'Origin'
```

→ 여러 origin 지원 — 반드시 `Vary: Origin` (캐시).

---

## 7. 세부 노트 — 이 폴더의 깊이

| 노트 | 영역 |
| --- | --- |
| [[same-origin-policy]] | SOP 의 정의 / 의미 / 예외 |
| [[simple-request]] | Simple request 의 조건 (Preflight 면제) |
| [[preflight]] | OPTIONS preflight 깊이 |
| [[cors-with-credentials]] | credentials: include + 함정 |
| [[cors-troubleshooting]] | 흔한 오류 / 디버깅 |

---

## 8. 흔한 오류 메시지 (Chrome)

```
Access to fetch at 'https://api.com/...' from origin 'https://app.com'
has been blocked by CORS policy:
- No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

→ 서버가 CORS 헤더 안 보냄.

```
- The 'Access-Control-Allow-Origin' header has a value 'https://other.com'
  that is not equal to the supplied origin.
```

→ Origin 매칭 X.

```
- The value of the 'Access-Control-Allow-Origin' header in the response
  must not be the wildcard '*' when the request's credentials mode is 'include'.
```

→ Wildcard + credentials 결합.

자세히 → [[cors-troubleshooting]]

---

## 9. CORS 가 X 인 경우

### 9.1 같은 origin
SOP 의 의미 — 같은 origin 은 자유.

### 9.2 `<img>`, `<script>`, `<link>` 태그
- 이미지 / 스크립트 / CSS — read 자체는 SOP X
- BUT — `<canvas>` 에 cross-origin 이미지 그리면 "tainted" 상태 → pixel data 읽기 X

### 9.3 `<iframe>`
- 표시는 OK
- JS 접근은 SOP

### 9.4 Form submit (전통)
- 양방향 X (CSRF 위험 ← SameSite cookie)
- 응답 읽기 X

### 9.5 Server-to-server
- CORS 는 **브라우저** 만 — 서버 fetch 는 무관

---

## 10. CORS 우회 — 주의

### 9.1 CORS Proxy (개발)
```
프록시 서버가 Allow-Origin: * 추가
브라우저 → proxy → 실제 API
```

→ 개발 / 데모 만. Production 보안 위험.

### 9.2 JSONP (옛, 사장)
- `<script>` 태그가 cross-origin 가능
- Callback 함수 명시
- 보안 위험 — XSS / 신뢰

### 9.3 nginx reverse proxy
```nginx
location /api/ {
    proxy_pass https://api.example.com/;
}
```

→ 같은 origin 으로 보이게.

자세히 → [[../../cdn-proxy/cdn-proxy]]

---

## 11. 면접 / 토픽

1. **SOP 와 CORS** — 왜 / 어떻게.
2. **Preflight 의 조건**.
3. **Wildcard + Credentials** 의 거부 이유.
4. **CORS 가 보안 메커니즘인가?** — 브라우저만 — 서버 보안 X.
5. **CORS vs CSRF** — 다른 개념.

---

## 12. 학습 자료

- **CORS — Fetch standard** — https://fetch.spec.whatwg.org/
- MDN CORS — https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS
- "CORS in 6 minutes" — web.dev
- "Same-origin policy" — MDN

---

## 13. 관련

- [[../http]] — HTTP hub
- [[same-origin-policy]] — SOP 기본
- [[simple-request]] / [[preflight]] / [[cors-with-credentials]] / [[cors-troubleshooting]]
- [[../methods/options]] — Preflight 의 OPTIONS
- [[../security/security]] — CSRF 비교
