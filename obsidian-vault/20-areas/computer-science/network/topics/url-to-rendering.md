---
title: "URL 입력 후 렌더링까지 — 네트워크 전반"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T10:05:00+09:00
tags:
  - network
  - browser
  - http
---

# URL 입력 후 렌더링까지 — 네트워크 전반

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 8 단계 / 자세히 |

**[[topics|↑ 토픽 hub]]** · **[[../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

`https://google.com` 입력 → 화면 표시 — **네트워크 / OS / 브라우저** 전반의 흐름. 면접 / 토론 단골.

---

## 2. 흐름 — 한눈

```
1. URL parsing
2. DNS 조회
3. TCP / TLS handshake
4. HTTP request
5. HTTP response
6. HTML parsing → DOM
7. CSS / JS 로딩 + 실행
8. Rendering (paint)
```

---

## 3. URL Parsing

```
https://www.google.com:443/search?q=test#section
↑       ↑               ↑   ↑       ↑         ↑
scheme  host            port path  query     fragment
```

### 브라우저
- Auto-completion / history
- IDN — punycode (예: münchen → xn--mnchen-3ya)
- Trailing slash 자동
- Scheme 추측 (default https)

---

## 4. DNS 조회

### 캐시 단계
```
Browser cache (1 분 정도)
        ↓ miss
OS cache (TTL)
        ↓ miss
hosts 파일
        ↓ miss
Recursive DNS (ISP / 8.8.8.8) — TTL
        ↓ miss
Root → TLD (.com) → Authoritative (google.com)
```

### 결과
- A record — IPv4
- AAAA record — IPv6 (있으면 우선, Happy Eyeballs)
- HTTPS record — HTTP/3 hint

자세히 → [[../dns/dns-hierarchy-resolution]] / [[../dns/dns-caching]]

### Anycast / GeoDNS
- 사용자 위치 별 다른 IP — CDN edge
- 자세히 → [[dns-load-balancing-anycast]]

---

## 5. TCP / TLS handshake

### TCP 3-way (1-RTT)
```
Client → Server: SYN
Client ← Server: SYN-ACK
Client → Server: ACK
```

### TLS 1.3 (1-RTT)
```
Client → Server: ClientHello + KeyShare
Client ← Server: ServerHello + KeyShare + Cert + Finished
Client → Server: Finished
                  (이후 데이터)
```

### 총
- HTTP/2 위 — 1 RTT (TCP) + 1 RTT (TLS) = 2 RTT
- HTTP/3 (QUIC) — 0-1 RTT (TCP 없음, TLS 와 결합)

자세히 → [[../tcp/tcp-handshake-3way]] / [[../tls-ssl/tls-1-3]]

### Session resumption
- 두 번째 접속 — 0-RTT 가능 (TLS PSK)

---

## 6. HTTP Request

```http
GET /search?q=test HTTP/2
Host: www.google.com
User-Agent: Mozilla/5.0 ...
Accept: text/html,application/xhtml+xml,...
Accept-Language: ko-KR
Accept-Encoding: gzip, deflate, br
Cookie: ...
Cache-Control: max-age=0
```

### 헤더
- Cookie (Authentication)
- Cache (If-None-Match / If-Modified-Since)
- Accept-Encoding (gzip / br)
- DNT (Do Not Track)

---

## 7. HTTP Response

```http
HTTP/2 200
Server: gws
Content-Type: text/html; charset=UTF-8
Content-Encoding: gzip
Cache-Control: private, max-age=0
Set-Cookie: ...
Strict-Transport-Security: max-age=31536000
X-XSS-Protection: 1; mode=block
Content-Security-Policy: ...

<html>... (gzipped)
```

### 응답 처리
- Decompress (gzip / br)
- Decoding (UTF-8)
- Cache 저장

---

## 8. HTML parsing → DOM

```
HTML bytes → tokenizer → tokens → tree (DOM)
```

### Preload scanner
- 본격 parsing 전 — <link>, <script>, <img> 미리 fetch

### Speculative parsing
- async / defer JS 가 막아도 — 다른 resource 미리

---

## 9. CSS / JS / 이미지 로딩

### CSS (render-blocking)
```html
<link rel="stylesheet" href="style.css">
```

→ 다운로드 / parse 까지 — paint 멈춤. CSSOM 생성.

### JS — 위치 / 옵션
```html
<script src="app.js"></script>             ← blocking (옛)
<script src="app.js" defer></script>       ← HTML parse 후 실행
<script src="app.js" async></script>       ← 다운로드 즉시 실행
<script type="module" src="app.js"></script>  ← async 와 비슷
```

### 이미지
- lazy loading: `<img loading="lazy">`
- WebP / AVIF (모던 포맷)

### Resource Hints
```html
<link rel="dns-prefetch" href="//other.com">
<link rel="preconnect" href="https://api.com">
<link rel="preload" href="font.woff2" as="font">
<link rel="prefetch" href="next-page.html">
```

자세히 → [[../dns/dns-caching]] (prefetch)

---

## 10. Critical Rendering Path

```
1. DOM (HTML)
2. CSSOM (CSS)
3. Render Tree (DOM + CSSOM)
4. Layout (위치 / 크기)
5. Paint (픽셀)
6. Composite (레이어 결합)
```

### 최적화
- Critical CSS inline
- Above-the-fold 우선
- JS 미루기 (defer)

---

## 11. JavaScript 실행

### Event loop
- Macro task / micro task
- Async / await — microtask

### Long tasks
- Main thread block → frame drop
- Worker / async chunk

---

## 12. 보안

### HSTS
- 처음 HTTPS 응답에 헤더 → 이후 모두 HTTPS

### Preload list
- 브라우저 내장 — 첫 방문도 HTTPS 강제

### Content-Security-Policy
- 어떤 리소스 허용

### SameSite cookie
- CSRF 방어

자세히 → [[../http/auth/auth]] (TBD)

---

## 13. 측정 — Web Vitals

| 메트릭 | 의미 |
| --- | --- |
| **TTFB** | Time to First Byte (네트워크) |
| **FCP** | First Contentful Paint |
| **LCP** | Largest Contentful Paint (2.5s good) |
| **CLS** | Cumulative Layout Shift (0.1 good) |
| **INP** | Interaction to Next Paint (200ms good) |
| **TBT** | Total Blocking Time |

### 도구
- Lighthouse (Chrome DevTools)
- PageSpeed Insights
- WebPageTest

---

## 14. HTTP/3 (QUIC) 의 차이

### TCP 위 → UDP 위
- TCP 없음 — connection 더 빠름
- HoL blocking 해결

### 0-RTT
- 재방문 — TLS handshake 안 함

### Connection migration
- WiFi ↔ LTE 전환 — connection 유지

자세히 → [[../quic/http-3]]

---

## 15. CDN 의 역할

```
사용자 → CDN edge (가까움) → cache hit → 즉시
                            cache miss → origin
```

- TLS 종료 (edge)
- 정적 리소스 캐시
- DDoS 흡수

자세히 → [[../cdn-proxy/cdn]]

---

## 16. 함정

### 함정 1 — DNS 의 stale
TTL 까지 옛 IP. 사용자 — DNS flush 필요.

### 함정 2 — HTTPS 리다이렉트의 RTT
HTTP → HTTPS — 추가 RTT. HSTS 권장.

### 함정 3 — Mixed content
HTTPS 페이지 — HTTP 리소스 block. 자동 upgrade (HTTPS).

### 함정 4 — Render-blocking JS
<head> 의 sync script — paint 막힘. defer / async.

### 함정 5 — Image 의 큰 크기
WebP / AVIF / srcset 활용.

### 함정 6 — 3rd party script
Analytics / Ads — 큰 부하. async / lazy.

---

## 17. 학습 자료

- "High Performance Browser Networking" (Grigorik)
- web.dev — Performance
- "Critical Rendering Path" (Google)
- Lighthouse / WebPageTest

---

## 18. 관련

- [[topics]] — Hub
- [[../dns/dns]] / [[../tcp/tcp]] / [[../tls-ssl/tls-ssl]]
- [[../http/http]] — HTTP hub
- [[../cdn-proxy/cdn]] — CDN
