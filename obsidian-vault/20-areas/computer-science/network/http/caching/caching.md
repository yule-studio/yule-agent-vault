---
title: "HTTP 캐싱 (Hub)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T22:10:00+09:00
tags:
  - network
  - http
  - caching
---

# HTTP 캐싱 (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | RFC 9111 — 캐시 hub |

**[[../http|↑ HTTP]]** · **[[../../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

**응답을 저장하고 재사용** 해 latency / 대역폭 / 서버 부하 감소. RFC 9111 (2022)
표준. 브라우저 / Proxy / CDN 의 핵심.

---

## 2. 캐시의 종류

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│ Browser │ ←→ │  Proxy  │ ←→ │   CDN   │ ←→ │ Reverse │ ←→ │ Origin  │
│  cache  │    │  cache  │    │ (Edge)  │    │  proxy  │    │ server  │
└─────────┘    └─────────┘    └─────────┘    └─────────┘    └─────────┘
   Private    Shared       Shared        Shared
```

### 종류
- **Private** — 한 사용자 (브라우저)
- **Shared** — 여러 사용자 (Proxy / CDN)

### Cache-Control 의 영향
- `public` — 모든 캐시 OK
- `private` — Private 만 (브라우저)
- `no-store` — 캐시 X

---

## 3. 캐시의 4 단계 흐름

```
요청 ─→ 캐시 검색
       ├ Hit + Fresh   → 캐시 응답
       ├ Hit + Stale   → Origin 검증 (조건부 GET)
       │                 ├ 304 Not Modified → 캐시 응답
       │                 └ 200 OK + 새 body → 캐시 갱신
       └ Miss          → Origin 요청 → 캐시 저장
```

---

## 4. Freshness — Fresh vs Stale

```
Fresh:  현재 시간 < (응답 시간 + max-age)
Stale:  지났음

Date: Tue, 13 May 2026 12:00:00 GMT
Cache-Control: max-age=300

→ 12:05:00 까지 Fresh
→ 12:05:01 부터 Stale
```

### Stale 도 사용 가능
- `stale-while-revalidate` — 배경 갱신
- `stale-if-error` — Origin 오류 시
- `must-revalidate` 면 X

---

## 5. 캐시 키

```
URL + Method + (Vary 헤더들)
```

### 예
```
GET /api/users + Accept-Encoding: gzip   → cache1
GET /api/users + Accept-Encoding: br     → cache2
GET /api/users + Authorization: ...      → 보통 캐시 X (개인)
```

### 커스텀 키 (CDN)
- Cloudflare — Custom cache key
- 쿼리 string 정렬 / 일부 무시

---

## 6. 세부 노트 — 이 폴더의 깊이

| 노트 | 영역 |
| --- | --- |
| [[cache-control]] | max-age / no-cache / no-store / public / private / immutable / s-maxage / stale-* |
| [[etag-conditional]] | ETag / If-None-Match / If-Match / 304 |
| [[last-modified]] | Last-Modified / If-Modified-Since / If-Unmodified-Since |
| [[vary-header]] | Vary 의 캐시 키 |
| [[cache-strategies]] | Cache-aside / Read-through / Write-through / Write-behind / Refresh-ahead |
| [[cdn-caching]] | Edge / Origin / Stale-While-Revalidate / Purge |

---

## 7. 일반적 캐시 정책 예

### 정적 자원 (이미지 / CSS / JS)
```http
Cache-Control: public, max-age=31536000, immutable
ETag: "v1.0-abc"
```

- 1 년 max-age
- `immutable` — 변경 안 됨
- 파일명에 hash (`/static/app.abc123.js`) — 변경 시 새 URL

### HTML (자주 변함)
```http
Cache-Control: public, max-age=60, must-revalidate
ETag: "v1"
```

- 1 분 max-age
- 만료 후 304 검증

### API 응답
```http
Cache-Control: private, max-age=0, must-revalidate
ETag: "v1"
```

또는:
```http
Cache-Control: no-cache
```

- 매번 304 검증
- private — 사용자별

### 민감 정보
```http
Cache-Control: no-store
```

- 캐시 절대 X
- 인증 후 응답, 결제 페이지

---

## 8. 캐시 무효화

### 자연 만료
- max-age 도달

### 명시적 무효화 (CDN)
- Cloudflare Purge API
- AWS CloudFront Invalidation
- Fastly Soft Purge / Hard Purge

### Cache Busting (브라우저)
```
/app.js?v=2          ← 쿼리 변경
/app-v2.js           ← 파일명 변경
/abc123/app.js       ← 경로 변경 (hash)
```

### CDN Tag Purge
```
Surrogate-Key: post-123 home-page
→ Purge by tag "post-123" → 모든 관련 URL 무효
```

---

## 9. 함정 — 흔한 캐시 사고

### 함정 1 — Cache-Control 없으면?
**Heuristic caching** — Last-Modified 의 10% 정도 캐시 (`Expires: now + (now - Last-Modified) × 0.1`).

→ 명시적 Cache-Control 권장.

### 함정 2 — 인증된 응답 캐시
`Authorization` 헤더 있으면 기본 캐시 X — 단, `Cache-Control: public` 명시 시 가능.

### 함정 3 — POST 캐시
기본 X — 명시 시만.

### 함정 4 — Vary: User-Agent
캐시 효율 폭락. 응용 / Edge logic 으로 분기.

### 함정 5 — Stale 응답
`stale-while-revalidate` 없으면 stale = miss.

### 함정 6 — CDN ↔ Origin 의 Cache-Control 차이
`s-maxage` 로 CDN 만 길게 캐시 가능 (브라우저는 짧게).

### 함정 7 — 잘못된 ETag
PUT/PATCH 응답에 ETag 갱신 누락 → 옛 ETag 사용.

### 함정 8 — Cookie 와 Vary
`Vary: Cookie` 안 하면 다른 사용자에게 캐시 누출 — 보안 사고.

---

## 10. 면접 / 토픽

1. **Cache-Control 의 max-age vs ETag**.
2. **304 Not Modified** — 어떻게 / 왜.
3. **public vs private**.
4. **Vary 의 역할**.
5. **CDN 의 stale-while-revalidate**.
6. **Cache Busting** 방법.
7. **인증된 응답의 캐시**.

---

## 11. 학습 자료

- **RFC 9111** (HTTP Caching, 2022)
- **High Performance Browser Networking** Ch. 11 (무료)
- MDN HTTP Caching — https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching
- Fastly / Cloudflare 캐시 가이드

---

## 12. 관련

- [[../http]] — HTTP hub
- [[../headers/general-headers]] — Cache-Control
- [[../headers/response-headers]] — ETag / Last-Modified / Vary
- [[../../cdn-proxy/cdn-proxy]] — CDN 캐싱
