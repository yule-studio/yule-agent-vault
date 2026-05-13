---
title: "Cache-Control 헤더"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T22:13:00+09:00
tags:
  - network
  - http
  - caching
  - cache-control
---

# Cache-Control 헤더

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 모든 directive 상세 |

**[[caching|↑ Caching]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

캐시 동작을 제어하는 **directive** 들의 모음. 응답 / 요청 양쪽 가능. RFC 9111.

---

## 2. 응답 Directives (서버 → 캐시)

### 2.1 max-age=N

```http
Cache-Control: max-age=3600       (1 시간)
```

- N 초 동안 fresh
- 가장 흔함

### 2.2 s-maxage=N

```http
Cache-Control: max-age=60, s-maxage=3600
```

- **공유 캐시만** (CDN / Proxy)
- max-age 와 별개로 적용
- 사용: CDN 1 시간, 브라우저 1 분

### 2.3 public

```http
Cache-Control: public, max-age=3600
```

- 누구나 캐시 OK (공유 캐시 포함)
- Authorization 헤더 있는 응답도 캐시 가능

### 2.4 private

```http
Cache-Control: private, max-age=300
```

- **브라우저만** 캐시 OK
- CDN / Proxy 캐시 X
- 사용자별 데이터

### 2.5 no-cache

```http
Cache-Control: no-cache
```

⚠️ **이름이 헷갈림** — "캐시 X" 가 아님:
- 캐시 가능 (저장 OK)
- **매번 origin 에 검증** (조건부 GET — 304 or 200)

→ "캐시 X" 는 `no-store`.

### 2.6 no-store

```http
Cache-Control: no-store
```

- **저장 X** — 모든 캐시 (메모리 / 디스크)
- 민감 정보 (결제, 인증, 의료)
- 응답이 사라지자마자 사라짐

### 2.7 must-revalidate

```http
Cache-Control: max-age=3600, must-revalidate
```

- 만료되면 **반드시** origin 검증
- 일반 캐시는 stale 사용 가능 — must-revalidate 면 X

### 2.8 proxy-revalidate

```http
Cache-Control: max-age=3600, proxy-revalidate
```

- 공유 캐시만 must-revalidate 같음

### 2.9 immutable

```http
Cache-Control: public, max-age=31536000, immutable
```

- "이 응답은 절대 변경 안 됨"
- 새로고침 (F5) 도 304 안 보냄 — 즉시 캐시
- 정적 자원 + hash 파일명에 권장

### 2.10 stale-while-revalidate=N (RFC 5861)

```http
Cache-Control: max-age=60, stale-while-revalidate=3600
```

- 만료 후 N 초 동안 stale 응답 OK
- 동시에 백그라운드로 origin 검증
- 사용자에게 빠른 응답 + 다음에 fresh

### 2.11 stale-if-error=N (RFC 5861)

```http
Cache-Control: max-age=60, stale-if-error=86400
```

- Origin 오류 (5xx / 네트워크) 시 stale 응답
- 1 일 동안 graceful degradation
- CDN 표준 기능

### 2.12 no-transform

```http
Cache-Control: no-transform
```

- 캐시 / Proxy 가 content 변경 X
- 모바일 ISP 의 자동 이미지 압축 / WebP 변환 방지

---

## 3. 요청 Directives (클라 → 캐시)

### 3.1 no-cache

```http
GET / HTTP/1.1
Cache-Control: no-cache
```

- "강제 origin 검증" (Ctrl+F5 새로고침)
- 클라가 캐시 fresh 도 origin 가도록

### 3.2 no-store

```http
Cache-Control: no-store
```

- 캐시에 저장 X (현 요청만)

### 3.3 max-age=N

```http
Cache-Control: max-age=0
```

- N 초 이내의 응답만 OK
- `max-age=0` = no-cache

### 3.4 max-stale=N

```http
Cache-Control: max-stale=300
```

- N 초까지의 stale 응답 OK
- "조금 오래된 거라도"

### 3.5 min-fresh=N

```http
Cache-Control: min-fresh=300
```

- 적어도 N 초 더 fresh 한 응답만

### 3.6 only-if-cached

```http
Cache-Control: only-if-cached
```

- 캐시에서만, 없으면 504 Gateway Timeout
- 오프라인 모드 / 측정

---

## 4. 조합 예

### 정적 자원
```http
Cache-Control: public, max-age=31536000, immutable
```

### HTML (변경 잦음)
```http
Cache-Control: public, max-age=60, must-revalidate
```

### API 응답 (캐시 + 검증)
```http
Cache-Control: private, max-age=0, must-revalidate
```

### 민감 정보
```http
Cache-Control: no-store
```

### CDN 만 길게
```http
Cache-Control: max-age=60, s-maxage=3600
```

### Stale-While-Revalidate
```http
Cache-Control: max-age=60, stale-while-revalidate=86400
```

---

## 5. 우선순위

여러 헤더 / directive 동시:

```
no-store > no-cache > must-revalidate > max-age
```

### 옛 vs 새
- **Cache-Control** 우선
- Expires / Pragma 는 호환성 fallback

---

## 6. 캐시 별 해석

| 캐시 | public | private | s-maxage |
| --- | --- | --- | --- |
| 브라우저 | ✅ | ✅ | ❌ (max-age 사용) |
| Proxy | ✅ | ❌ | ✅ |
| CDN | ✅ | ❌ | ✅ |

---

## 7. Authorization 헤더 영향

기본:
- `Authorization` 있는 요청의 응답 = **공유 캐시 X**

예외 — Cache-Control 명시:
- `public`
- `must-revalidate`
- `s-maxage`

→ 인증 후 응답이지만 공유 캐시 가능.

---

## 8. 디버깅

### Chrome DevTools — Network
- Response Headers 의 Cache-Control 확인
- "Size" 컬럼 — `(from disk cache)` / `(from memory cache)`
- "Disable cache" 체크 (개발)

### curl

```bash
curl -I https://example.com/api/users
# Cache-Control: max-age=300
```

### 캐시 무효화 트릭 (개발)
```bash
curl -H "Cache-Control: no-cache" -H "Pragma: no-cache" ...
```

---

## 9. CDN 별 차이

### Cloudflare
- `Cache-Control` 기본 따름
- "Cache Everything" Page Rule 가능
- `CDN-Cache-Control` (separate)

### AWS CloudFront
- Origin 의 Cache-Control 사용
- Behavior 의 TTL 설정 가능

### Fastly
- `Surrogate-Control` 헤더 (CDN 전용)
- Soft Purge

---

## 10. 함정

### 함정 1 — no-cache vs no-store
no-cache 는 저장 OK + 매번 검증. no-store 는 저장 X. **헷갈림**.

### 함정 2 — max-age=0 vs no-cache
사실상 같음 — 둘 다 매번 검증 (다른 응답이 fresh 면 사용).

### 함정 3 — Authorization 응답의 캐시
명시 안 하면 캐시 X. 의도된 공유 캐시면 public 명시.

### 함정 4 — public 의 의미
- 모든 status code 캐시 가능 (보통 200/206/301/410 만 캐시되지만 명시 시 더)
- `private` 의 반대 — 공유 캐시 OK

### 함정 5 — immutable 의 강한 캐시
잘못 설정 시 사용자가 새 버전 안 받음. hash 파일명과 함께.

### 함정 6 — stale-while-revalidate 미지원
일부 옛 브라우저 / CDN. Fallback (max-age 짧게).

### 함정 7 — no-transform 누락
모바일 ISP 가 이미지 변환 — 화질 깨짐.

---

## 11. 학습 자료

- **RFC 9111** Section 5.2 (Cache-Control)
- **RFC 5861** (stale-while-revalidate, stale-if-error)
- MDN Cache-Control
- "HTTP caching" — web.dev

---

## 12. 관련

- [[caching]] — Caching hub
- [[etag-conditional]] — 검증
- [[last-modified]] — 검증
- [[vary-header]] — 캐시 키
- [[../headers/general-headers]] — Cache-Control 위치
