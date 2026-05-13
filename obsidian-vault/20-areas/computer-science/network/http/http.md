---
title: "HTTP — Hub (전체 hub)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:00:00+09:00
tags:
  - network
  - http
  - hub
---

# HTTP — Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | HTTP 전체 hub + 12 sub-folder |

**[[../network|↑ network hub]]** · **[[../osi-7-layer/layer-7-application/layer-7-application|↑↑ L7 Application]]**

> HTTP 는 너무 큰 주제 — sub-folder 로 깊이 분리.

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **계층** | L7 Application |
| **포트** | 80 (HTTP), 443 (HTTPS / HTTP/3) |
| **표준 단체** | IETF (RFC), W3C |
| **출처** | Tim Berners-Lee, CERN 1989 |
| **현재 RFC** | RFC 9110-9114 (2022) |

---

## 1. 한 줄 정의

**Hyper**Text **T**ransfer **P**rotocol — 클라이언트-서버 간 데이터 교환 표준.
WWW (1989) 의 토대. 현재 인터넷 트래픽의 압도적 다수.

---

## 2. HTTP 의 본질

```
요청 (Request) ← 메서드 + URL + 헤더 + 본문
        ↓
응답 (Response) ← 상태 코드 + 헤더 + 본문
```

- **Stateless** — 매 요청 독립 (쿠키 / 세션 / 토큰으로 상태 흉내)
- **Text-based** (HTTP/1) / **Binary-framed** (HTTP/2+)
- **Request-response** 모델
- **Extensible** — 헤더 추가로 확장

---

## 3. Sub-folder 인덱스

### 3.1 [[versions/versions|↗ versions/]] — 버전 진화

| 노트 | 영역 |
| --- | --- |
| [[versions/versions]] | hub + 비교 표 |
| [[versions/http-0-9]] | 1991 — 첫 HTTP |
| [[versions/http-1-0]] | 1996 — RFC 1945 |
| [[versions/http-1-1]] | 1997-2014 — RFC 7230~7235 / RFC 9110-9112 |
| [[versions/http-2]] | 2015 — RFC 7540 → RFC 9113 |
| [[versions/http-3-quic]] | 2022 — RFC 9114 (QUIC 위) |
| [[versions/version-comparison]] | 버전별 비교 / negotiation |

### 3.2 [[methods/methods|↗ methods/]] — HTTP 메서드

| 노트 | 영역 |
| --- | --- |
| [[methods/methods]] | hub |
| [[methods/get]] | GET — 조회 |
| [[methods/post]] | POST — 생성 / 처리 |
| [[methods/put]] | PUT — 전체 갱신 |
| [[methods/patch]] | PATCH — 부분 갱신 |
| [[methods/delete]] | DELETE — 삭제 |
| [[methods/head]] | HEAD — 헤더만 |
| [[methods/options]] | OPTIONS — 허용 메서드 / CORS preflight |
| [[methods/trace-connect]] | TRACE / CONNECT |
| [[methods/idempotency-safety]] | 멱등 / 안전 / 캐시 가능 |

### 3.3 [[status-codes/status-codes|↗ status-codes/]] — 상태 코드

| 노트 | 영역 |
| --- | --- |
| [[status-codes/status-codes]] | hub |
| [[status-codes/1xx-informational]] | 100/101/102/103 |
| [[status-codes/2xx-success]] | 200/201/202/204/206 |
| [[status-codes/3xx-redirection]] | 301/302/303/304/307/308 |
| [[status-codes/4xx-client-errors]] | 400/401/403/404/405/409/410/418/422/429 |
| [[status-codes/5xx-server-errors]] | 500/502/503/504/505 |

### 3.4 [[headers/headers|↗ headers/]] — 헤더

| 노트 | 영역 |
| --- | --- |
| [[headers/headers]] | hub + 분류 |
| [[headers/general-headers]] | Date / Connection / Cache-Control / Via |
| [[headers/request-headers]] | Host / User-Agent / Accept / Referer / Origin |
| [[headers/response-headers]] | Server / Location / WWW-Authenticate |
| [[headers/entity-headers]] | Content-Type / Content-Length / Content-Encoding |
| [[headers/content-negotiation]] | Accept / Accept-Language / Accept-Encoding / Vary |
| [[headers/custom-x-headers]] | X-Forwarded-For / X-Request-ID / X- prefix 정책 |

### 3.5 [[caching/caching|↗ caching/]] — 캐싱

| 노트 | 영역 |
| --- | --- |
| [[caching/caching]] | hub |
| [[caching/cache-control]] | max-age / no-cache / no-store / private / public / s-maxage |
| [[caching/etag-conditional]] | ETag / If-None-Match / If-Match / 304 |
| [[caching/last-modified]] | Last-Modified / If-Modified-Since |
| [[caching/vary-header]] | Vary 의 의미 |
| [[caching/cache-strategies]] | Cache-aside / Read-through / Write-through 등 |
| [[caching/cdn-caching]] | Edge cache / Origin / Stale-while-revalidate |

### 3.6 [[cookies/cookies|↗ cookies/]] — 쿠키

| 노트 | 영역 |
| --- | --- |
| [[cookies/cookies]] | hub |
| [[cookies/cookie-attributes]] | Domain / Path / Expires / Max-Age / Secure / HttpOnly / SameSite / Partitioned |
| [[cookies/cookie-security]] | XSS / CSRF 방어와 결합 |
| [[cookies/session-cookies]] | Session ID / 서버 상태 |
| [[cookies/jwt-vs-cookie]] | JWT 토큰 vs 세션 쿠키 |

### 3.7 [[cors/cors|↗ cors/]] — CORS

| 노트 | 영역 |
| --- | --- |
| [[cors/cors]] | hub |
| [[cors/same-origin-policy]] | SOP 의 정의 / 의미 |
| [[cors/simple-request]] | 단순 요청 조건 |
| [[cors/preflight]] | OPTIONS preflight |
| [[cors/cors-with-credentials]] | credentials: include |
| [[cors/cors-troubleshooting]] | 흔한 오류 / 디버깅 |

### 3.8 [[rest/rest|↗ rest/]] — REST

| 노트 | 영역 |
| --- | --- |
| [[rest/rest]] | hub |
| [[rest/rest-principles]] | 6 가지 제약 (Fielding 2000) |
| [[rest/resource-design]] | URI 설계 / 리소스 모델 |
| [[rest/hateoas]] | Hypermedia / Richardson Maturity Model |
| [[rest/api-versioning]] | URL / 헤더 / Media type |
| [[rest/rest-vs-rpc-graphql]] | 비교 |

### 3.9 [[security/security|↗ security/]] — HTTP 보안 헤더

| 노트 | 영역 |
| --- | --- |
| [[security/security]] | hub |
| [[security/hsts]] | Strict-Transport-Security |
| [[security/csp]] | Content-Security-Policy |
| [[security/x-frame-options]] | Clickjacking 방어 |
| [[security/x-content-type-options]] | MIME sniffing 방어 |
| [[security/referrer-policy]] | Referer 노출 제어 |
| [[security/permissions-policy]] | Feature-Policy 후속 |

### 3.10 [[performance/performance|↗ performance/]] — 성능

| 노트 | 영역 |
| --- | --- |
| [[performance/performance]] | hub |
| [[performance/keep-alive]] | Connection: keep-alive (1.1) |
| [[performance/pipelining]] | HTTP/1.1 의 (실패한) pipelining |
| [[performance/compression-encoding]] | gzip / br / zstd / deflate |
| [[performance/transfer-encoding]] | chunked / Content-Encoding 차이 |
| [[performance/range-requests]] | Range / 206 Partial Content |

### 3.11 [[streaming/streaming|↗ streaming/]] — 스트리밍

| 노트 | 영역 |
| --- | --- |
| [[streaming/streaming]] | hub |
| [[streaming/chunked-transfer]] | Transfer-Encoding: chunked |
| [[streaming/server-sent-events]] | SSE / EventSource |
| [[streaming/websocket]] | WebSocket (HTTP Upgrade) |
| [[streaming/http2-push]] | Server Push (deprecated) |
| [[streaming/long-polling]] | 옛 비동기 패턴 |

### 3.12 [[auth/auth|↗ auth/]] — 인증

| 노트 | 영역 |
| --- | --- |
| [[auth/auth]] | hub |
| [[auth/basic-digest]] | HTTP Basic / Digest Auth |
| [[auth/bearer-token]] | Authorization: Bearer (OAuth/JWT) |
| [[auth/oauth2-flow]] | OAuth 2.0 grant types |
| [[auth/api-keys]] | API Key 패턴 |
| [[auth/mtls-cert-auth]] | 클라이언트 인증서 |

---

## 4. HTTP 요청 / 응답 예

### 요청
```http
GET /api/users/123 HTTP/1.1
Host: api.example.com
User-Agent: curl/7.68.0
Accept: application/json
Authorization: Bearer eyJhbGc...
Accept-Encoding: gzip, br

```

### 응답
```http
HTTP/1.1 200 OK
Date: Tue, 13 May 2026 12:00:00 GMT
Server: nginx/1.25.0
Content-Type: application/json; charset=UTF-8
Content-Length: 134
Cache-Control: max-age=300
ETag: "abc123"

{"id":123,"name":"Alice","email":"alice@example.com"}
```

---

## 5. HTTP 의 9 가지 핵심 개념

1. **메서드** (Method / Verb) — GET, POST, PUT, ...
2. **URL** — 자원 식별 (scheme://host:port/path?query#frag)
3. **상태 코드** — 1xx-5xx
4. **헤더** — 메타데이터
5. **본문** (Body) — 실제 데이터
6. **MIME 타입** — Content-Type
7. **인증** — Basic/Bearer/OAuth/mTLS
8. **세션** — 쿠키 / JWT
9. **캐싱** — Cache-Control / ETag

---

## 6. 면접 / 큰 질문

1. **GET vs POST** — 멱등 / 안전 / 캐시.
2. **3-way handshake + HTTPS** — TCP + TLS + HTTP.
3. **HTTP/1.1 → HTTP/2 → HTTP/3 의 진화**.
4. **Cookie vs Session vs JWT**.
5. **CORS 가 왜 / 어떻게**.
6. **REST 의 제약**.
7. **캐싱 (ETag / Cache-Control / 304)**.
8. **HSTS / CSP**.

---

## 7. 학습 자료

- **RFC 9110 (Semantics)** / **9111 (Caching)** / **9112 (HTTP/1.1)** / **9113 (HTTP/2)** / **9114 (HTTP/3)** — 2022 통합
- **HTTP: The Definitive Guide** (Gourley)
- **High Performance Browser Networking** (Grigorik) — 무료
- **MDN HTTP** — https://developer.mozilla.org/en-US/docs/Web/HTTP
- **Web Almanac** (HTTP Archive) — 실측 통계

---

## 8. 관련

- [[../tls-ssl/tls-ssl]] — HTTPS 의 기반
- [[../tcp/tcp]] — HTTP/1/2 의 transport
- [[../quic/quic]] — HTTP/3 의 transport
- [[../dns/dns]] — URL → IP
- [[../osi-7-layer/layer-7-application/layer-7-application]] — L7 hub
