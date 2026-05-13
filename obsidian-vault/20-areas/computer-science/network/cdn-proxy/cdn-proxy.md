---
title: "CDN / Proxy — Hub"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T07:00:00+09:00
tags:
  - network
  - cdn
  - proxy
---

# CDN / Proxy — Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | CDN / Forward / Reverse |

**[[../network|↑ network hub]]** · **[[../osi-7-layer/osi-7-layer|↑↑ OSI]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **계층** | L7 (HTTP) 위주 |
| **목적** | 캐시 / 분산 / 보안 / 익명 |
| **유형** | CDN / Forward Proxy / Reverse Proxy |

---

## 1. 한 줄 정의

**중간자** — 클라이언트와 서버 사이에서 캐시 / 라우팅 / 보안 / 변환.

---

## 2. 3 가지 — 한눈

| 유형 | 목적 | 위치 |
| --- | --- | --- |
| **CDN** | 정적 콘텐츠 캐시 / 가속 | edge (전 세계) |
| **Forward Proxy** | 클라이언트 보호 / 익명 | 클라이언트 측 |
| **Reverse Proxy** | 서버 보호 / 라우팅 | 서버 측 |

자세히 → [[cdn]], [[forward-proxy]], [[reverse-proxy]]

---

## 3. 흐름

### CDN
```
사용자 → edge (가까운 도시) → cache hit → 응답
                            cache miss → origin → 응답 + cache
```

### Forward Proxy
```
클라이언트 → 회사/학교 proxy → 인터넷
            (전체 트래픽 통과 — 필터 / 로그)
```

### Reverse Proxy
```
인터넷 → reverse proxy → 내부 서버 1
                       → 내부 서버 2
                       → 내부 서버 3
```

---

## 4. CDN — Content Delivery Network

### 정의
- 정적 / 동적 콘텐츠를 **사용자 가까운 edge 에서 제공**
- Akamai (1998), Cloudflare (2009), Fastly (2011), CloudFront (2008)

### 효과
- Latency ↓ (지역 가까이)
- Origin 부하 ↓ (캐시)
- DDoS 방어 (트래픽 흡수)
- TLS termination
- WAF / Bot 차단

자세히 → [[cdn]]

---

## 5. Forward Proxy

### 정의
- 클라이언트 측 — 클라이언트가 명시적 사용
- 회사 / 학교 / VPN

### 사용
- 컨텐츠 필터 (성인 / 도박 차단)
- 트래픽 로깅
- 익명 (Tor)
- 캐시 (옛 — Squid)

자세히 → [[forward-proxy]]

---

## 6. Reverse Proxy

### 정의
- 서버 측 — 클라이언트는 reverse proxy 만 알고 백엔드 모름
- Nginx / HAProxy / Envoy / Cloudflare / Apache

### 사용
- Load balancing
- TLS termination
- WAF
- 캐시
- 압축 (gzip / br)
- HTTP/2 / HTTP/3 — 백엔드는 HTTP/1.1

자세히 → [[reverse-proxy]]

---

## 7. CDN 이 Reverse Proxy 의 특수

```
CDN = 분산 Reverse Proxy + 캐시 + DDoS + ...
```

### 차이
- Reverse proxy — 한 곳 (data center)
- CDN — 전 세계 edge 수백 곳

---

## 8. Forward vs Reverse — 한눈

| | Forward | Reverse |
| --- | --- | --- |
| **목적** | 클라이언트 보호 / 익명 | 서버 보호 / 분산 |
| **누가 안다** | 클라이언트 (설정) | 클라이언트 (해당 도메인) |
| **누가 모른다** | 서버 (real client 모름) | 클라이언트 (real server 모름) |
| **위치** | 클라이언트 측 | 서버 측 |
| **예** | Squid, Tor, 회사 proxy | Nginx, ALB, Cloudflare |

---

## 9. 세부 노트

| 노트 | 영역 |
| --- | --- |
| [[cdn]] | CDN |
| [[forward-proxy]] | Forward Proxy |
| [[reverse-proxy]] | Reverse Proxy |
| [[edge-computing]] | Edge / Workers |

---

## 10. 학습 자료

- "Akamai Internet Report"
- Cloudflare / Fastly 블로그
- Nginx / Envoy docs

---

## 11. 관련

- [[../http/caching/caching]] — HTTP 캐시
- [[../load-balancing/load-balancing]] — Reverse proxy 와 결합
- [[../dns/dns]] — CDN 의 DNS 라우팅
