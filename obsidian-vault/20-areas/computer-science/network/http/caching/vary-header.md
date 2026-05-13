---
title: "Vary 헤더 — 캐시 키"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T22:26:00+09:00
tags:
  - network
  - http
  - caching
  - vary
---

# Vary 헤더 — 캐시 키

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Vary 의 의미 / 캐시 효율 / 함정 |

**[[caching|↑ Caching]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

응답이 **어느 요청 헤더에 따라 달라지는지** 명시. 캐시가 키에 포함.

---

## 2. 동작

```http
GET / HTTP/1.1
Accept-Encoding: gzip

HTTP/1.1 200 OK
Content-Encoding: gzip
Vary: Accept-Encoding
{...gzipped...}

→ 캐시 키 = URL + Accept-Encoding=gzip
```

```http
GET / HTTP/1.1
Accept-Encoding: br

HTTP/1.1 200 OK
Content-Encoding: br
Vary: Accept-Encoding
{...brotli...}

→ 캐시 키 = URL + Accept-Encoding=br  (다른 캐시 entry)
```

---

## 3. 자주 쓰는 Vary

### 3.1 Vary: Accept-Encoding

```http
Vary: Accept-Encoding
```

- gzip / br / 압축 X 별 다른 응답
- **가장 흔함** — 모든 압축 응답에

### 3.2 Vary: Accept-Language

```http
Vary: Accept-Language
```

- 언어별 다른 컨텐츠
- 다국어 사이트

### 3.3 Vary: Accept

```http
Vary: Accept
```

- JSON / XML / HTML 별 다른 응답
- Content Negotiation

### 3.4 Vary: Origin

```http
Vary: Origin
```

- CORS — Origin 별 다른 Allow-Origin 응답
- 캐시가 다른 Origin 에 잘못된 응답 안 돌려줌

### 3.5 Vary: Cookie

```http
Vary: Cookie
```

- 로그인 / 사용자 별 다른 컨텐츠
- ⚠️ 매우 비효율 — 거의 캐시 불가능
- → `Cache-Control: private` 권장

### 3.6 여러 헤더

```http
Vary: Accept-Encoding, Accept-Language
```

- 콤마 구분 — 모두 캐시 키에

---

## 4. Vary 의 문제 — Vary: User-Agent

```http
Vary: User-Agent
```

### 문제
- UA 가 거의 무한 변형
  - Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/...
  - Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/...
  - Mozilla/5.0 (iPhone; CPU iPhone OS 15_0...) ...
  - curl/7.68.0
  - 봇 종류 수만

→ 각 UA 별 다른 캐시 entry — **사실상 캐시 무효**.

### 해결
- UA 분기는 응용 / Edge logic (Cloudflare Workers) 에서
- 결과를 명시적 캐시 키로 변환 (`X-Device: mobile/desktop`)
- Vary: User-Agent 는 피하기

---

## 5. Vary: *

```http
Vary: *
```

- **모든** 요청 헤더에 따라 다름
- 사실상 캐시 X
- 의도: "캐시하지 마"
- 실제: `Cache-Control: no-store` 가 명확

→ Vary: * 사용 X.

---

## 6. CDN 별 Vary 처리

### Cloudflare
- 기본 Vary 무시 (성능)
- "Respect Existing Headers" 옵션 활성 시 Vary 따름
- Cache Key Custom 설정 권장

### Fastly
- Vary 따름
- 복잡한 Vary 는 캐시 효율 ↓

### AWS CloudFront
- Behavior 의 "Cache Based on Selected Request Headers" 와 결합
- 명시한 헤더만 키에

---

## 7. Edge / Origin 의 Vary 정책

```
Origin → Vary: Accept-Encoding, Accept-Language
   ↓
Edge CDN
   ↓ 캐시 key:
   /page + Accept-Encoding=gzip + Accept-Language=ko
```

### 정규화 (Normalization)
- `Accept-Encoding: gzip, br, deflate` → 정렬 / 일관
- 단순화: `gzip, br` 만 인식, 나머지 묶음
- Cloudflare Workers / VCL 로 구현

---

## 8. Browser 의 Vary 처리

브라우저:
- 캐시 entry 마다 응답의 Vary 헤더 저장
- 다음 요청 시 같은 Vary 헤더 매칭 검증
- 다르면 cache miss

### Vary: User-Agent 의 브라우저 영향
- 한 사용자는 항상 같은 UA → 한 entry 만 사용 — OK
- CDN 에서만 문제

---

## 9. 함정

### 함정 1 — Vary: User-Agent
거의 무한 entry. 매우 안 좋음.

### 함정 2 — Vary: *
캐시 무력. no-store 사용.

### 함정 3 — Vary 누락 (압축)
같은 URL 의 gzip / 비압축 캐시 혼동 → 클라가 압축 헤더 없는데 gzip 받음 → 깨짐.

### 함정 4 — Vary 와 동적 응답
같은 헤더지만 응용이 비결정적 응답 — 캐시 일관성 깨짐.

### 함정 5 — Vary: Cookie 의 비효율
사용자별 cookie → 거의 매번 miss. `private` + 짧은 max-age.

### 함정 6 — Origin 헤더와 CORS
`Vary: Origin` 안 하면 — 한 Origin 의 응답이 다른 Origin 에 캐시 누출 → CORS 깨짐.

### 함정 7 — 정규화 안 함
`Accept-Encoding: gzip, br` vs `Accept-Encoding: br, gzip` — 같은 의미지만 다른 캐시 entry.

---

## 10. 학습 자료

- **RFC 9110** Section 12.5.5 (Vary)
- **RFC 9111** Section 4.1
- Cloudflare "Vary Header"
- MDN Vary

---

## 11. 관련

- [[caching]] — Caching hub
- [[cache-control]]
- [[../headers/content-negotiation]] — Accept-* 와 함께
- [[cdn-caching]] — CDN 의 Vary 처리
