---
title: "라우팅 (Routing) — Hub"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T19:25:00+09:00
tags:
  - network
  - routing
  - bgp
  - ospf
---

# 라우팅 (Routing) — Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 라우팅 hub + 6 프로토콜 노트 |

**[[../network|↑ network hub]]** · **[[../osi-7-layer/layer-3-network/layer-3-network|↑↑ L3 Network]]**

> 라우팅 자료구조 (테이블 / FIB / Longest Prefix) 는
> [[../osi-7-layer/layer-3-network/ip-routing-basics|↗ IP 라우팅 기초]]. 이 폴더는
> **라우팅 프로토콜** 의 깊이.

---

## 1. 한 줄 정의

라우팅 프로토콜 = 라우터들이 **자동으로 경로를 학습 / 교환** 하는 규약. 정적
라우팅의 대안 — 큰 망의 변화에 적응.

---

## 2. 분류

### 2.1 IGP vs EGP

| 분류 | 영역 | 예 |
| --- | --- | --- |
| **IGP** (Interior) | 한 AS 내부 | RIP, OSPF, IS-IS, EIGRP |
| **EGP** (Exterior) | AS 간 | **BGP** |

**AS** (Autonomous System) — ISP / 큰 조직 단위. ASN (1-4 byte) 으로 식별.

### 2.2 동작 방식

| 방식 | 학습 | 예 |
| --- | --- | --- |
| **Distance Vector** | 이웃에게서 들음 | RIP, EIGRP |
| **Link State** | 토폴로지 전체 broadcast | OSPF, IS-IS |
| **Path Vector** | 경로 자체 + policy | BGP |
| **Hybrid** | DV+LS 절충 | EIGRP |

### 2.3 정적 vs 동적

| 기준 | 정적 | 동적 |
| --- | --- | --- |
| 설정 | 수동 | 자동 학습 |
| 변경 적응 | X | O |
| 부하 | 0 | CPU/대역폭 일부 |
| 적합 | 작은 / 변화 없는 | 큰 / 변화 잦은 |

---

## 3. 주요 프로토콜 한눈

| 프로토콜 | 분류 | 메트릭 | AD (Cisco) | 표준 |
| --- | --- | --- | --- | --- |
| **RIP v1/v2** | DV / IGP | Hop count | 120 | RFC 2453 |
| **RIPng** | DV / IGP IPv6 | Hop count | 120 | RFC 2080 |
| **OSPF v2** | LS / IGP | Cost (BW 기반) | 110 | RFC 2328 |
| **OSPFv3** | LS / IGP IPv6 | Cost | 110 | RFC 5340 |
| **IS-IS** | LS / IGP | Cost | 115 | ISO 10589 |
| **EIGRP** | Hybrid / IGP | Composite (BW/Delay) | 90 / 170 | RFC 7868 (Cisco) |
| **BGP** | Path / EGP | Path attributes | 20 / 200 | RFC 4271 |
| **MP-BGP** | BGP 확장 | — | — | RFC 4760 |

---

## 4. 라우팅 프로토콜의 핵심 메커니즘

### 4.1 Discovery — 이웃 발견
- Hello packet (OSPF, EIGRP, IS-IS)
- BGP — 수동 설정

### 4.2 Topology Exchange
- DV: 이웃 의 라우팅 테이블 받음
- LS: 모든 라우터의 link 정보 broadcast → 자기가 SPF 계산
- Path: 전체 경로 + policy attributes

### 4.3 Path Selection
- 최소 hop / cost / policy
- Tie-breaking 규칙

### 4.4 Convergence
- 변화 후 모든 라우터의 정보 일치까지 걸리는 시간
- 빠를수록 좋음 (수 초 ↔ 분)

### 4.5 Loop Prevention
- **Split Horizon** — 받은 인터페이스로 광고 X
- **Poison Reverse** — 받은 인터페이스로는 infinity 광고
- **Hold-Down** — 변화 후 일정 시간 광고 무시
- **Maximum Hop** — RIP 15 hop

---

## 5. 라우팅 결정 우선순위 (Cisco AD)

여러 프로토콜에서 같은 prefix 학습 시:

```
Connected (0) → Static (1) → eBGP (20) → EIGRP (90)
→ OSPF (110) → IS-IS (115) → RIP (120) → External EIGRP (170)
→ iBGP (200) → Unknown (255)
```

낮은 AD 우선. 같은 AD 면 metric.

---

## 6. 세부 노트 — 이 폴더의 깊이

| 노트 | 영역 |
| --- | --- |
| [[rip]] | RIPv1/v2, RIPng, 30s Hello, 16 hop |
| [[ospf]] | Link State, LSA, Area, DR/BDR, SPF Dijkstra |
| [[bgp]] | Path Vector, AS Path, attributes, iBGP/eBGP, Route reflector, Confederation |
| [[eigrp]] | Cisco Hybrid, DUAL 알고리즘, K1-K5 |
| [[mpls]] | Label switching, LDP, RSVP-TE, L3VPN, EVPN |
| [[pim-multicast]] | PIM-DM, PIM-SM, RP, SPT, MSDP |

---

## 7. 라우팅 정책 (Policy)

### 7.1 Route Map
- 광고 / 수신 시 prefix 필터링
- AS Path 조작
- Metric 조정

### 7.2 Prefix List
- ACL 의 라우팅 버전
- BGP 에서 흔히

### 7.3 Communities (BGP)
- 64-bit 태그
- "이 prefix 는 backup 만 / Asia 만 광고" 등

---

## 8. 라우팅 보안

### 8.1 Authentication
- **TCP-MD5** (BGP, RFC 2385) — deprecated
- **TCP-AO** (RFC 5925) — 모던
- **OSPF MD5 / SHA**
- **IS-IS HMAC-SHA**

### 8.2 BGP 사고
- **BGP Hijacking** — 잘못된 prefix 광고 → 트래픽 가로채기
- **Route Leak** — 정책 위반 광고
- 방어: **RPKI** (RFC 6480) + ROA, BGPsec

### 8.3 ARP Spoofing
- L2 — L3 와 별개. DAI 로 방어.

---

## 9. 면접 / 토픽

1. **OSPF vs BGP** — IGP/EGP, LS/PV.
2. **BGP path selection** 규칙 (Local Pref / AS Path / MED).
3. **RIP 의 한계** (16 hop / convergence 느림).
4. **OSPF LSA 종류**.
5. **MPLS 의 의의** — L2/L3 사이.
6. **BGP Hijack** — Facebook 2021 등.
7. **iBGP full mesh / Route Reflector**.
8. **Convergence time** 비교.

---

## 10. 학습 자료

- **Routing TCP/IP Vol.1 & 2** (Doyle / Carroll) — Cisco 정석
- **BGP** (Iljitsch van Beijnum)
- **OSPF and IS-IS** (Jeff Doyle)
- **Internet Routing Architectures** (Sam Halabi)
- Cisco / Juniper / Arista 공식 문서
- RIPE / APNIC / NANOG 자료

---

## 11. 관련

- [[../osi-7-layer/layer-3-network/ip-routing-basics]] — 자료구조
- [[../osi-7-layer/layer-3-network/multicast-igmp]] — PIM 의 IGMP
- [[../ip/ip|↑ IP 폴더]]
- [[../osi-7-layer/layer-3-network/layer-3-network]] — L3 hub
