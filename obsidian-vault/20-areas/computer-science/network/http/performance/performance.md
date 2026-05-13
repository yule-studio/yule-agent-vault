---
title: "HTTP 성능 (Hub)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T00:35:00+09:00
tags:
  - network
  - http
  - performance
---

# HTTP 성능 (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Keep-Alive / Pipelining / 압축 / chunked / Range |

**[[../http|↑ HTTP]]** · **[[../../network|↑↑ network hub]]**

---

## 1. HTTP 성능의 5 차원

1. **연결 재사용** — Keep-Alive, HTTP/2 multiplexing
2. **압축** — gzip / br / zstd
3. **캐싱** — Cache-Control / ETag / CDN
4. **부분 전송** — Range request, chunked
5. **병렬화** — HTTP/2 stream, HTTP/3 QUIC

---

## 2. 세부 노트

| 노트 | 영역 |
| --- | --- |
| [[keep-alive]] | Persistent connection (HTTP/1.1) |
| [[pipelining]] | HTTP/1.1 의 (실패한) pipelining |
| [[compression-encoding]] | gzip / br / zstd / deflate |
| [[transfer-encoding]] | chunked / TE vs CE |
| [[range-requests]] | 부분 다운로드 / 206 |

---

## 3. 핵심 메트릭 — Web Vitals

| 메트릭 | 의미 |
| --- | --- |
| **TTFB** (Time to First Byte) | 첫 byte 도착 시간 |
| **FCP** (First Contentful Paint) | 첫 콘텐츠 표시 |
| **LCP** (Largest Contentful Paint) | 가장 큰 요소 표시 (Core Web Vital) |
| **FID** (First Input Delay) | 첫 입력 응답 |
| **INP** (Interaction to Next Paint) | 모든 입력 응답 (FID 후속) |
| **CLS** (Cumulative Layout Shift) | 레이아웃 흔들림 |

### Core Web Vitals (Google)
- LCP < 2.5s
- INP < 200ms
- CLS < 0.1

### Server-Timing (응답 헤더)
```http
Server-Timing: db;dur=53.2, cache;dur=2.1, total;dur=58.7
```

---

## 4. 클라 측 최적화

### Resource Hints
```html
<link rel="dns-prefetch" href="https://api.example.com">
<link rel="preconnect" href="https://api.example.com">
<link rel="preload" href="/critical.css" as="style">
<link rel="prefetch" href="/next-page.html">
<link rel="modulepreload" href="/app.js">
```

### Async / Defer
```html
<script src="..." async></script>
<script src="..." defer></script>
```

### Lazy loading
```html
<img src="..." loading="lazy">
<iframe src="..." loading="lazy"></iframe>
```

---

## 5. 서버 측 최적화

### 응답 압축
```http
Content-Encoding: br
```

### Cache 헤더
```http
Cache-Control: public, max-age=31536000, immutable
ETag: "v1"
```

### 103 Early Hints
```http
HTTP/1.1 103 Early Hints
Link: </style.css>; rel=preload; as=style
```

### HTTP/2 / HTTP/3
- 멀티플렉싱
- HPACK / QPACK 헤더 압축

---

## 6. CDN

- 사용자 가까운 Edge
- 정적 자원 + 동적 짧은 캐시
- Origin Shielding

자세히 → [[../caching/cdn-caching]]

---

## 7. 면접 / 토픽

1. **HTTP/1.1 vs HTTP/2 vs HTTP/3** 성능 차이.
2. **HoL Blocking** — TCP vs QUIC.
3. **gzip vs Brotli** — 차이.
4. **Range Request** — 어떤 상황.
5. **Keep-Alive** vs **Multiplexing**.

---

## 8. 학습 자료

- **High Performance Browser Networking** (Grigorik) — 무료
- web.dev — Performance
- Web Almanac — 실측 통계

---

## 9. 관련

- [[../http]] — HTTP hub
- [[keep-alive]], [[pipelining]], [[compression-encoding]], [[transfer-encoding]], [[range-requests]]
- [[../versions/version-comparison]] — 버전별 성능
- [[../caching/caching]]
