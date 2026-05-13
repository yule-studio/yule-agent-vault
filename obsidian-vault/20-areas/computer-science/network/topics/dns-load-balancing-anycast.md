---
title: "DNS Load Balancing / Anycast"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T10:30:00+09:00
tags:
  - network
  - dns
  - load-balancing
  - anycast
---

# DNS Load Balancing / Anycast

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | DNS-LB / Geo / Anycast |

**[[topics|↑ 토픽 hub]]** · **[[../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

DNS 응답을 **상황 따라 다르게** — 지역 / 부하 / 헬스에 따라 사용자 별 best IP.

---

## 2. 방법 — 한눈

| 방법 | 설명 |
| --- | --- |
| **Round-robin DNS** | 여러 A record 순환 |
| **Weighted** | 가중치 따라 비율 |
| **Geo DNS** | 사용자 IP → 가장 가까운 |
| **Latency DNS** | RUM / probe 기반 빠른 곳 |
| **Health-check DNS** | 죽은 backend 응답 X |
| **Anycast** | BGP — 같은 IP, 가까운 라우터 |

---

## 3. Round-Robin DNS

### 동작
```
example.com  IN  A  1.2.3.4
example.com  IN  A  5.6.7.8
example.com  IN  A  9.10.11.12
```

→ 매 응답 — 순서 회전.

### 효과
- 단순 / 무료
- 클라이언트 — 첫 IP 시도

### 한계
- DNS cache — 한 IP pin 가능
- 죽은 backend 도 응답 (health check X)
- 부하 차이 무시

---

## 4. Weighted DNS

```
Route 53 / Cloud DNS — weight 설정
example.com → 70% IP A, 30% IP B
```

### 사용
- Canary (90 → 10)
- Blue-green deployment
- 점진 마이그레이션

---

## 5. Geo DNS

### 동작
- 사용자 IP — GeoIP DB 매핑
- 사용자 위치 별 다른 응답

### 예
```
Asia 사용자 → tokyo.example.com (1.2.3.4)
US 사용자 → us-east.example.com (5.6.7.8)
EU 사용자 → eu-west.example.com (9.10.11.12)
```

### 제품
- AWS Route 53 Geolocation
- Cloudflare Geo-Steering
- NS1
- DNSMadeEasy

### 한계
- DNS resolver 의 IP 봄 (사용자 IP X)
- EDNS Client Subnet (RFC 7871) — 사용자 IP 일부 forward
- 일부 ISP — EDNS X → resolver IP 만

---

## 6. Latency / RUM DNS

### 정의
- RUM (Real User Monitoring) — 실제 사용자의 latency 측정
- 최저 latency 의 region 응답

### 제품
- NS1 Pulsar
- Cedexis (Citrix → Cisco)

### 효과
- Geo 보다 정확 (실측)
- 네트워크 사고 자동 대응

---

## 7. Health-check DNS

### 동작
```
Backend A — health probe — OK → DNS 응답 포함
Backend B — health probe — FAIL → 응답 제외
```

### 제품
- AWS Route 53 Health Check
- Cloud DNS Health Check

### 한계
- TTL — fail 후 갱신 까지 사용자 다른 곳 못 감
- 짧은 TTL (30-60s)

---

## 8. Anycast

### 정의
- 같은 IP 를 **여러 위치에서 BGP announce**
- 라우터 가 가장 가까운 곳으로 routing

### 차이
| | Unicast | Multicast | Anycast |
| --- | --- | --- | --- |
| **수신자** | 한 명 | 그룹 | 한 명 (best of N) |
| **사용** | 일반 | IGMP / streaming | DNS / CDN |

### 흐름
```
Client → 1.2.3.4 (Cloudflare DNS)
        ↓ BGP routing
        가까운 PoP 의 1.2.3.4 (East Asia)

다른 사용자 (Europe) → 1.2.3.4 → European PoP
```

### 효과
- 자동 (BGP 가 routing)
- 빠름 (Geo 보다 정확)
- Failover (PoP 다운 → BGP withdrawal → 다음 PoP)

---

## 9. Anycast — 사례

### DNS
- Root server (a-m.root-servers.net)
- 1.1.1.1 (Cloudflare)
- 8.8.8.8 (Google)
- 9.9.9.9 (Quad9)

### CDN
- Cloudflare (edge 모두 같은 IP)
- Fastly
- Akamai (일부)
- AWS Global Accelerator

### Load Balancer
- Maglev (Google) — 여러 LB 같은 IP
- Katran (Facebook)

자세히 → [[../load-balancing/l4-load-balancing]]

---

## 10. Anycast 의 동작

### BGP
- 같은 prefix `192.0.2.0/24` 를 여러 AS / 위치에서 announce
- 라우터 — best path 선택 (AS path 짧음 / preference)

### 한계
- **Stateful 통신 어려움** — 같은 IP 라도 다른 PoP 도착 가능
- TCP — 일관성 어려움 (다른 PoP 의 다른 state)
- DNS / UDP — 상태 없음 — 친화

### TCP 해결
- LB / proxy 가 connection 흡수
- 한 user 의 5-tuple → 같은 PoP (BGP 안정 가정)

---

## 11. EDNS Client Subnet (ECS)

### 문제
- Public resolver (8.8.8.8) — 사용자 IP 안 보임
- Geo DNS — 부정확

### ECS (RFC 7871)
```
Recursive resolver → Authoritative:
"사용자의 /24 prefix: 1.2.3.0/24"
```

→ Authoritative 가 더 정확하게.

### 프라이버시
- 사용자 위치 노출 (대략 /24)
- 1.1.1.1 (Cloudflare) — ECS 안 함 (프라이버시 우선)
- 8.8.8.8 — ECS 함

### 결과
- CDN — ECS 친화 (정확한 region)
- 프라이버시 친화 resolver — ECS X

---

## 12. DNS 의 한계

### 캐시
- TTL — 짧으면 부하, 길면 변경 늦음
- 클라이언트 cache 무시 가능

### 분산
- 한 도메인 — 모든 사용자 같은 응답 (resolver 의 다음 분기)
- 정밀 routing 어려움

### Anycast 가 보완
- IP 자체가 분산 (BGP)
- DNS cache 의 부작용 X

---

## 13. Multi-CDN — DNS 활용

### 정의
- 여러 CDN — DNS 가 분기
- 사용자 별 최적 CDN

### 도구
- NS1 Pulsar
- Cedexis
- AWS Route 53

### 흐름
```
사용자 → DNS 쿼리
        ↓
NS1 — 사용자 IP / RUM data 분석
        ↓
"이 사용자 — Cloudflare 빠름" → Cloudflare IP 응답
또는 "Akamai 빠름" → Akamai IP
```

---

## 14. 함정

### 함정 1 — TTL 너무 짧음
DNS 부하 ↑. 60s 권장 (failover 가능 + 캐시 합리).

### 함정 2 — DNS cache 무력
일부 OS / 브라우저 — TTL 무시. 클라이언트 별 다름.

### 함정 3 — Anycast 의 stateful
TCP / UDP-with-state — PoP 변경 시 끊김. LB 가 흡수.

### 함정 4 — Resolver 의 위치
8.8.8.8 사용자 — Geo DNS 가 Google 위치 봄 (가까운 곳 응답).
ECS 없으면 부정확.

### 함정 5 — 인터넷 사고 시 BGP 흐름
BGP convergence — 수 분. Anycast 가 자동이지만 빈틈.

### 함정 6 — DNSSEC + Geo
DNSSEC 의 서명된 응답 + 다른 응답 — 검증 깨짐. CDN 가 자체 키 관리.

---

## 15. 학습 자료

- Cloudflare 블로그 (anycast)
- "Anycast: Where We Are and Where We Are Going" (Sigcomm)
- RFC 7871 (EDNS Client Subnet)
- NS1 / Cedexis docs

---

## 16. 관련

- [[topics]] — Hub
- [[../dns/dns]] / [[../dns/dns-hierarchy-resolution]]
- [[../cdn-proxy/cdn]] — Anycast 의 응용
- [[../load-balancing/l4-load-balancing]] — Maglev / Katran
