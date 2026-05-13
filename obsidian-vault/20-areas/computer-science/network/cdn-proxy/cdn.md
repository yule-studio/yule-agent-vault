---
title: "CDN — Content Delivery Network"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T07:05:00+09:00
tags:
  - network
  - cdn
  - caching
  - edge
---

# CDN — Content Delivery Network

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Edge / PoP / Cache / Origin Shield |

**[[cdn-proxy|↑ CDN/Proxy]]** · **[[../network|↑↑ network hub]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **시작** | Akamai (1998) |
| **주요** | Cloudflare / Akamai / Fastly / CloudFront / Google CDN |
| **계층** | L7 + DNS + Anycast |

---

## 1. 한 줄 정의

**전 세계 edge 서버에 캐시** — 사용자 가까이에서 콘텐츠 제공. Latency / 부하 / 보안 해결.

---

## 2. CDN 의 효과

### Latency ↓
```
사용자 (서울) → origin (미국 서부)        — 200 ms
사용자 (서울) → CDN edge (서울)            — 10 ms
```

### Origin 부하 ↓
- Cache hit ratio 90%+ — origin 으로 가는 트래픽 1/10
- DDoS 흡수

### TLS termination
- Edge 가 TLS 처리 — origin 평문 (또는 내부 TLS)

### 비용 ↓
- Origin egress (data out) 절감 — CDN 이 대부분 처리

---

## 3. PoP — Point of Presence

### 정의
- CDN edge 서버가 모인 곳 (data center / 작은 시설)
- 한 도시에 여러 PoP

### 규모
- Cloudflare — 300+ 도시
- Akamai — 1000+
- Fastly — 90+
- CloudFront — 600+ edge / 13 regional cache

---

## 4. CDN 의 흐름

```
사용자 → DNS 조회 → CDN 의 IP (anycast)
                              ↓
사용자 → 가까운 PoP edge
                              ↓
Edge: cache lookup
  Hit  → 즉시 응답 (95% 케이스)
  Miss → Regional cache (Origin Shield) 조회
              Hit  → 응답 + edge 저장
              Miss → Origin 조회 → 응답 + 저장
```

### Cache hit ratio
- 정적 (이미지 / JS / CSS) — 95-99%
- HTML — 60-80%
- 동적 — 0-30%

---

## 5. DNS / Anycast

### Anycast
- 같은 IP 를 전 세계 PoP 에 광고 (BGP)
- 라우터 가 가까운 PoP 로

### 옛 — GeoDNS
- 사용자 IP 보고 가까운 PoP 의 IP 응답
- 예: GeoIP DB

### 모던 — Anycast (Cloudflare)
- BGP 의 자연스러운 라우팅
- Failover 자동 (BGP 갱신)

자세히 → [[../topics/dns-load-balancing-anycast]]

---

## 6. 캐시 — Cache key

### 기본
```
cache_key = scheme + host + path + query
```

### 변형
- **Vary** — Accept-Encoding, Accept-Language
- **Cookie / Header 제외** — query 만 변형
- **Strip query** — UTM 등 추적 파라미터 제거

### TTL
```
Cache-Control: public, max-age=3600
```

### Purge / Invalidation
- API — `purge /path`
- 전체 / 태그 / Surrogate-Key

---

## 7. Origin Shield

### 정의
- Edge 와 origin 사이의 추가 계층
- 모든 edge → Origin Shield → origin

### 효과
- Origin 의 cache miss 부담 ↓ (edge 가 많아도 shield 만 origin 으로)
- Cache hit ratio ↑

### 제품
- CloudFront — Origin Shield (선택)
- Fastly — Shielding
- Akamai — Tiered Distribution

---

## 8. 정적 vs 동적

### 정적
- HTML / CSS / JS / 이미지 / 비디오
- 모든 사용자 동일 → cache 친화

### 동적
- 사용자 별 (로그인 / 개인화)
- 옛 — CDN 우회
- 모던 — Edge compute 로 처리 (Lambda@Edge, Workers)

### Dynamic Content Acceleration
- Origin 연결 최적화 (TCP optimization, persistent connection)
- TLS 종료 (edge)
- HTTP/2 / HTTP/3
- 헤더 압축 / 변환

---

## 9. ESI — Edge Side Includes

### 정의
- HTML 의 일부 — edge 에서 가져옴
- 페이지 fragment 캐싱

```html
<esi:include src="/header" />
<esi:include src="/cart" />     ← 동적 (사용자 cart)
```

- header — 캐시 가능
- cart — 매번 origin

### 사용
- Akamai, Varnish
- 모던 — SSR + ISR (Next.js) 가 대체

---

## 10. Image / Video 최적화

### 이미지
- WebP / AVIF 자동 변환 (디바이스 지원 보고)
- 크기 변환 (`image.jpg?w=300`)
- 압축

### 비디오
- HLS / DASH 적응형 스트리밍
- 비트레이트 변환
- 자동 자막

### 제품
- Cloudflare Images / Stream
- Fastly Image Optimizer
- AWS CloudFront + MediaConvert

---

## 11. DDoS 방어

### Volumetric (L3/L4)
- Anycast 가 트래픽 분산
- 수백 Gbps → 각 PoP 가 일부씩

### Application (L7)
- WAF
- Bot 차단 (JavaScript challenge, CAPTCHA)
- Rate limit

### Cloudflare 사례
- 2024 — 5.6 Tbps DDoS 흡수

---

## 12. WAF / Bot Management

### WAF
- OWASP Top 10
- 커스텀 규칙

### Bot
- Fingerprint (TLS / HTTP / JS)
- ML 분류 — good bot (Googlebot) vs bad bot
- CAPTCHA / managed challenge

---

## 13. Edge Compute (Workers / Lambda@Edge)

자세히 → [[edge-computing]]

### 핵심
- Edge 에서 JS / WASM 실행
- A/B testing / 개인화 / auth / rewrite
- Latency ms 단위

---

## 14. Multi-CDN

### 정의
- 여러 CDN 사용 (Akamai + Cloudflare + ...)
- DNS / RUM 기반 routing
- 한 CDN 장애 시 — 다른 CDN

### 도구
- NS1 Pulsar
- Cedexis (Citrix)
- AWS Route 53 weighted

### 효과
- 가용성 ↑
- 협상 leverage
- Region 별 최적 CDN

---

## 15. CDN 선택 — 기준

| 기준 | 강자 |
| --- | --- |
| **글로벌 PoP 많음** | Cloudflare / Akamai |
| **저렴** | Cloudflare (free tier) / Bunny.net |
| **개발자 친화** | Fastly / Cloudflare |
| **엔터프라이즈** | Akamai |
| **AWS 통합** | CloudFront |
| **비디오** | Akamai / Fastly |
| **Edge compute** | Cloudflare Workers / Fastly Compute / Vercel |

---

## 16. CDN 의 한계 / 함정

### 함정 1 — 동적 콘텐츠
캐시 어려움. ESI / Edge compute 필요.

### 함정 2 — Cookie / 개인화
사용자 별 응답 — cache key 분리 → hit ratio ↓.

### 함정 3 — Purge 지연
일부 CDN — 30 초+ 지연. Soft purge / surrogate key.

### 함정 4 — Origin IP 노출
DNS 의 origin record / 메일 헤더 / 옛 IP 의 cache.
방어 — origin 의 firewall 가 CDN IP 만 허용.

### 함정 5 — TLS cert 관리
CDN 의 cert + origin 의 cert. ACM / Cloudflare Origin Cert.

### 함정 6 — Hot key
한 path 의 매우 큰 트래픽 → 한 edge 부담. Origin shield.

### 함정 7 — Region 누락 / overload
PoP 적은 region (아프리카 일부) — latency.

### 함정 8 — Cost 의 데이터 전송
Big CDN egress — 비싸짐. Bunny / BunnyCDN / Backblaze + Cloudflare.

---

## 17. 학습 자료

- "High Performance Browser Networking" (Grigorik)
- Cloudflare / Fastly / Akamai 블로그
- "The Tangled Web" (Zalewski)
- web.dev — CDN 최적화

---

## 18. 관련

- [[cdn-proxy]] — Hub
- [[reverse-proxy]] — CDN 의 부모 개념
- [[edge-computing]] — Workers / Lambda@Edge
- [[../http/caching/caching]] — HTTP 캐시
- [[../dns/dns]] — Anycast
