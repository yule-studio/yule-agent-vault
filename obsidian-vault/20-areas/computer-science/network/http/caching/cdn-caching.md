---
title: "CDN 캐싱 (Edge / Origin)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T22:35:00+09:00
tags:
  - network
  - http
  - caching
  - cdn
---

# CDN 캐싱 (Edge / Origin)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Edge / Purge / Surrogate-Key / Soft Purge |

**[[caching|↑ Caching]]** · **[[../http|↑↑ HTTP]]**

> CDN 자체 / 인프라 는 [[../../cdn-proxy/cdn-proxy|↗ CDN/Proxy]]. 이 노트는 **HTTP 캐싱** 의 CDN 측면.

---

## 1. 한 줄 정의

CDN = **사용자 가까운 Edge 서버의 분산 캐시**. Origin 의 부하 ↓, latency 50-200ms → 5-20ms.

---

## 2. CDN 의 캐싱 흐름

```
사용자 → Edge (가까운 위치, 캐시 확인)
   ├ Hit  → 즉시 응답 (Origin 안 거침)
   └ Miss → Origin 조회 → 캐시 저장 → 응답
```

### 다단계 캐시
```
사용자 → Edge → Tier-1 cache → Tier-2 (region) → Origin
```

Cloudflare 의 "Tiered Cache", Fastly 의 Shielding.

---

## 3. CDN 의 캐시 키

기본:
```
key = (Host, Path)
```

확장:
- Query string 일부 / 전체
- Cookie 일부
- Header (Vary)
- Geo (지역)
- Device (mobile/desktop)

### 정규화 (Normalization)
- `?a=1&b=2` vs `?b=2&a=1` → 같은 캐시 키
- 일부 query 무시 (`utm_source=...` 광고)

---

## 4. Cache-Control vs CDN-Cache-Control

### 기본 — 같은 Cache-Control
```http
Cache-Control: public, max-age=300, s-maxage=3600
```

- 브라우저 5 분
- CDN 1 시간

### 분리 (RFC 9213, 2022)
```http
Cache-Control: max-age=300                   ← 브라우저
CDN-Cache-Control: max-age=3600              ← CDN
Surrogate-Control: max-age=86400             ← Fastly proprietary
```

→ Edge / Browser 다른 정책 명시.

---

## 5. Surrogate-Key (Fastly / VCL)

```http
HTTP/1.1 200 OK
Surrogate-Key: post-123 user-42 home
```

### 사용
- 응답에 여러 태그 부착
- Purge 시 태그로 무효화

```bash
# Fastly API
curl -X POST "https://api.fastly.com/service/SERVICE_ID/purge/post-123" \
  -H "Fastly-Key: $KEY"

# → post-123 태그된 모든 URL 무효
```

### Cloudflare 의 Cache Tags
- 비슷한 기능
- `Cache-Tag: post-123, home` 헤더

---

## 6. Purge 종류

### 6.1 Soft Purge

```
Edge 에서 stale 표시 — 다음 요청 시 Origin 검증
stale-while-revalidate 와 결합 — 사용자에게 즉시 stale 응답 + 백그라운드 갱신
```

→ Fastly Soft Purge.

### 6.2 Hard Purge

```
Edge 에서 즉시 삭제
다음 요청 = Origin 직접
```

→ 큰 부하 일으킬 수 있음 (cache stampede).

### 6.3 Wildcard / Prefix Purge

```
PURGE /assets/*
```

- 디렉토리 전체
- 일부 CDN 만 지원

### 6.4 Tag Purge

```
PURGE TAG post-123
```

- Surrogate-Key / Cache-Tag

---

## 7. Stale-While-Revalidate (CDN 핵심)

```http
Cache-Control: max-age=60, stale-while-revalidate=86400
```

### 동작
```
1. max-age=60 동안 fresh — 그대로 응답
2. 만료 후 86400 초 (1 일):
   - 사용자에게 stale 즉시 응답
   - 동시에 origin 검증 (백그라운드)
   - 검증 결과로 캐시 갱신
3. 86400 후 stale 사용 X — origin 직접
```

### 효과
- 사용자 latency 일정
- Origin 부하 분산

---

## 8. Stale-If-Error

```http
Cache-Control: max-age=60, stale-if-error=86400
```

### 동작
```
Origin 이 5xx 또는 timeout
→ 1 일 동안 stale 응답 사용 (graceful degradation)
→ 사용자에게는 정상으로 보임
```

### 효과
- Origin 장애 시 사용자 영향 최소화
- "오늘은 영상이 못 올라왔지만 어제 영상은 보임" 같은 패턴

---

## 9. Edge Workers / Functions

### Cloudflare Workers
```javascript
addEventListener("fetch", event => {
  const url = new URL(event.request.url)
  
  // 캐시 키 커스터마이즈
  const cacheKey = new Request(`${url.origin}${url.pathname}`, event.request);
  
  // 응답 조작
  event.respondWith(
    caches.default.match(cacheKey).then(r =>
      r || fetch(event.request).then(r2 => {
        caches.default.put(cacheKey, r2.clone());
        return r2;
      })
    )
  );
});
```

### AWS Lambda@Edge / CloudFront Functions
- 비슷한 능력
- Viewer Request / Origin Request / Origin Response / Viewer Response

### Fastly VCL
- Varnish Configuration Language
- 더 강력 (오래된 패러다임)

---

## 10. Origin Shielding

```
사용자 → Edge (수백 위치)
            ↓
       Shield (1 region)
            ↓
         Origin
```

### 동기
- Edge 1000 곳 → Origin 1000 miss 동시
- Shield 가 중간 캐시 → Origin 부하 ↓

### 사용
- Cloudflare Tiered Cache
- Fastly Shielding
- Argo Smart Routing

---

## 11. CDN 의 동적 콘텐츠

### 동적 = 캐시 X?
옛 — 정적 (HTML/CSS/JS/이미지) 만 캐시.

### 모던
- API 응답도 짧게 캐시 (5-60 초)
- Edge logic 으로 personalization (cookie / geo)
- Cloudflare Workers + KV (Edge Storage)

### Example — 짧은 API 캐시
```http
Cache-Control: public, max-age=10, stale-while-revalidate=60
```

- 10 초 fresh — 인기 API 부하 ↓
- 60 초 stale — 일관성 약간 손실 (수용 가능)

---

## 12. 인증 + CDN

```http
[로그인 페이지]
Cache-Control: no-store

[공개 페이지]
Cache-Control: public, max-age=300

[사용자 콘텐츠]
Cache-Control: private, max-age=60     ← CDN 캐시 X
```

### Auth + CDN 패턴
- 정적 자원 → CDN 캐시
- 인증 페이지 → Origin (no-cache)
- API 의 hot key → 짧은 캐시 + Vary

---

## 13. 함정

### 함정 1 — CDN 의 stale 응답
사용자가 Old version 봄. Purge / Versioning.

### 함정 2 — Cache stampede
TTL 만료 후 다수 동시 miss. Origin Shielding / Stale-While-Revalidate.

### 함정 3 — Authorization 의 CDN 처리
`Authorization` 있는 응답 — CDN 기본 캐시 X. `public` 명시.

### 함정 4 — Query string 변형
`?utm_source=...` 마다 다른 캐시 entry — 정규화 필요.

### 함정 5 — Vary 의 폭발
`Vary: Cookie` 면 캐시 무효.

### 함정 6 — CDN-Cache-Control 미지원
일부 CDN 만. `s-maxage` 가 fallback.

### 함정 7 — Soft / Hard Purge 혼동
Soft → stale 유지. Hard → 즉시 origin 부하.

### 함정 8 — Region 별 다른 응답
지역 별 다른 컨텐츠 — Edge logic 으로 분기. Vary: CF-IPCountry 등.

---

## 14. 학습 자료

- **RFC 9111** (Caching)
- **RFC 9213** (CDN-Cache-Control)
- Cloudflare "Cache Everything" / Tiered Cache
- Fastly "Cache control tutorial"
- "Caching at scale" — Cloudflare blog

---

## 15. 관련

- [[caching]] — Caching hub
- [[cache-control]] — directive
- [[vary-header]] — 캐시 키
- [[cache-strategies]] — 일반 패턴
- [[../../cdn-proxy/cdn-proxy]] — CDN 인프라
