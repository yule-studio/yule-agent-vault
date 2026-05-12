---
title: "MPLS — Multi-Protocol Label Switching"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T19:50:00+09:00
tags:
  - network
  - mpls
  - l3vpn
  - sr
---

# MPLS — Multi-Protocol Label Switching

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Label / LDP / RSVP-TE / L3VPN / SR / EVPN |

**[[routing|↑ Routing]]** · **[[../network|↑↑ network hub]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **계층** | "L2.5" — L2 와 L3 사이 |
| **EtherType** | 0x8847 (unicast), 0x8848 (multicast) |
| **라벨 크기** | 20 bit (1M label) |
| **RFC** | 3031, 3032, 4364 (L3VPN), 8402 (SR) |
| **출처** | 1990s 후반 (Tag Switching, Cisco) |

---

## 1. 한 줄 정의

**Label 기반 forwarding** — IP 헤더 대신 짧은 label 로 빠르게 전송. 통신사 백본 /
기업 WAN 의 표준. L2VPN, L3VPN, Traffic Engineering 의 기반.

---

## 2. 왜 MPLS — 동기

### 2.1 1990s 의 라우터 성능
- IP forwarding 은 매번 longest prefix match (느림)
- ATM 의 fixed-size cell forwarding 이 빠름
- → "ATM 의 속도 + IP 의 유연성" → MPLS

### 2.2 현대 의의 (HW lookup 빠른 후)
- **Traffic Engineering** — 명시 경로
- **L2/L3 VPN** — 고객 분리
- **Fast Reroute** — 50ms 이내 회복

---

## 3. MPLS Label 헤더

```
+--------+--------+--------+--------+
|       Label (20)     | EXP | S | TTL|
| 0                  19 | 3 | 1 | 8 |
+--------+--------+--------+--------+
```

- **Label** (20 bit) — forwarding 결정 (1M label)
- **EXP** (3 bit) — QoS (Traffic Class)
- **S** (1 bit) — Bottom of Stack (1 이면 마지막 label)
- **TTL** (8 bit) — IP TTL 와 비슷

### Label Stack
한 패킷에 여러 label (push / pop):
```
[Outer Label][Inner Label][...][IP][TCP][Data]
```

L3VPN 의 경우 2 label (외부 transport + 내부 VPN).

---

## 4. MPLS Forwarding

```
Edge LSR (Ingress) → P (transit) → P → ... → Edge LSR (Egress)

Ingress: IP 패킷 받음 → label push → label-based forwarding
P (Provider): label swap
Egress: label pop → IP forwarding (또는 PHP — Penultimate Hop Pop)
```

### 동사
- **Push** — label 추가
- **Swap** — label 교환
- **Pop** — label 제거

### LFIB (Label Forwarding Information Base)
- IP 의 FIB 와 같은 자료구조
- (incoming label, interface) → (outgoing label, interface) 매핑

---

## 5. Label 분배 — LDP / RSVP-TE

### 5.1 LDP — Label Distribution Protocol (RFC 5036)
- IGP 와 동기 — IGP 가 알려주는 경로에 label 할당
- 단순, 자동
- TE 불가

### 5.2 RSVP-TE — Resource Reservation Protocol (RFC 3209)
- 명시 경로 + 대역폭 예약
- Traffic Engineering
- 복잡

### 5.3 BGP-Labeled Unicast (RFC 8277)
- BGP 가 label 분배
- Inter-AS, L3VPN

### 5.4 Segment Routing (RFC 8402, 2017)
- LDP / RSVP-TE 폐기
- IGP 의 확장 (OSPF / IS-IS) 으로 label 분배
- 모던 표준

---

## 6. MPLS L3VPN (RFC 4364)

ISP 의 가장 큰 사용 사례:

```
고객 A site 1 ─ PE-A1 ─ MPLS backbone ─ PE-A2 ─ 고객 A site 2
                                                 (10.0.0.0/24)

PE 라우터:
- VRF (VPN Routing & Forwarding) — 고객별 라우팅 테이블
- MP-BGP 로 VPNv4 prefix 교환 (RD: 64512:1, RT: 64512:1)
- 두 label: outer (transport, PE→PE), inner (VPN, label = VRF)
```

### 6.1 RD — Route Distinguisher
- 8 byte (보통 ASN:Number)
- 같은 IP (10.0.0.0/24) 가 여러 고객에 있어도 RD 로 구분

### 6.2 RT — Route Target
- BGP Community 형식
- import / export 정책
- "이 prefix 를 어느 VRF 에 import 할까?"

### 6.3 VPNv4 Address Family

MP-BGP 가 (RD + IPv4) = VPNv4 광고:
```
RD:IPv4 prefix
64512:1:10.0.0.0/24
```

---

## 7. MPLS L2VPN

L2 frame 을 MPLS 위에:

### 7.1 VPWS (Virtual Private Wire Service)
- Point-to-Point — 두 PE 사이 가상 link
- "Pseudowire"

### 7.2 VPLS (Virtual Private LAN Service)
- Multipoint — 여러 PE 가 하나의 가상 LAN
- 옛 표준, 복잡

### 7.3 EVPN — Ethernet VPN (RFC 7432)
- VPLS 의 후속
- BGP control plane
- Active-Active multi-homing
- 데이터센터 표준 (VXLAN + EVPN)

---

## 8. Segment Routing (SR / SRv6)

### 8.1 SR-MPLS
- MPLS label 위 SR
- Segment List 로 명시 경로
- LDP / RSVP-TE 대체

### 8.2 SRv6 (Segment Routing IPv6)
- MPLS label 대신 IPv6 주소 (SID)
- IPv6 Extension Header (SRH)
- 단순화 — 한 컨트롤 plane

### 8.3 사용
- 5G / 모바일 백본
- Cloud / DC interconnect
- 의도 기반 (Intent-Based) 라우팅

---

## 9. Traffic Engineering

### 9.1 LSP (Label Switched Path)
- 명시 경로 — Headend 라우터가 정의
- 대역폭, 우선순위, affinity 등 제약

### 9.2 RSVP-TE
- 신호화로 LSP 수립
- IGP-TE (OSPF / IS-IS 의 TE 확장) 의 정보 활용

### 9.3 FRR (Fast Reroute)
- backup LSP 미리 준비
- 장애 시 < 50 ms 복구

---

## 10. PHP — Penultimate Hop Pop

```
Egress PE 의 부담 줄이기:
- 마지막 P 라우터가 outer label pop
- Egress 는 inner label / IP 만 처리
```

→ HW 의 IP / MPLS lookup 분리.

---

## 11. EXP / QoS

```
EXP 3 bit = 8 priority class
DSCP (IP) ↔ EXP mapping
```

통신사 SLA 의 기반.

---

## 12. MPLS vs SDN

### 12.1 옛 (전통)
- 라우터 분산 결정 (IGP + LDP)
- 정책 변경 어려움

### 12.2 SDN + MPLS
- Controller 가 LSP 결정
- BGP-LS 로 controller 가 토폴로지 학습
- PCEP (Path Computation Element Protocol) 으로 LSP 설정

### 12.3 모던 — SR 기반 SDN
- SR 로 stateless TE
- Controller 가 segment list 만 push

---

## 13. Cisco 설정 예 (L3VPN)

```
! PE 라우터
ip vrf CUSTOMER_A
  rd 64512:1
  route-target import 64512:1
  route-target export 64512:1

interface Gi0/1
  ip vrf forwarding CUSTOMER_A
  ip address 10.0.0.1 255.255.255.0

router bgp 64512
  address-family vpnv4
    neighbor 1.1.1.1 activate
    neighbor 1.1.1.1 send-community extended
  address-family ipv4 vrf CUSTOMER_A
    redistribute connected
    redistribute static
```

---

## 14. 디버깅

```
show mpls forwarding-table
show mpls ldp neighbor
show mpls traffic-eng tunnels
show ip vrf
show ip route vrf CUSTOMER_A
show bgp vpnv4 unicast all
```

---

## 15. 함정

### 함정 1 — MTU 영향
MPLS label 4 byte 마다 추가 → MTU 줄어듦. PMTUD 어려움 (MPLS 가 ICMP X).

### 함정 2 — 잘못된 RT
import / export 잘못되면 VPN 트래픽 누수 — 보안 사고.

### 함정 3 — LDP 와 IGP 동기 부재
LDP 가 IGP 보다 느리면 일시 black hole. Synchronization 기능 활성.

### 함정 4 — Single LSR 의 LFIB 가득
대규모 ISP — 별도 라우터 / 하드웨어 필요.

### 함정 5 — SR-MPLS / LDP 공존 복잡
점진 마이그레이션의 어려움.

### 함정 6 — MPLS 의 외부 노출 X
디버깅 어려움 — looking glass 외부서 안 보임.

---

## 16. 학습 자료

- RFC 3031 (MPLS architecture), RFC 4364 (L3VPN), RFC 7432 (EVPN), RFC 8402 (SR)
- **MPLS-Enabled Applications** (Ina Minei)
- **MPLS Fundamentals** (Luc De Ghein)
- **Segment Routing Part I / II** (Filsfils)
- Cisco / Juniper MPLS 가이드

---

## 17. 관련

- [[bgp]] — MP-BGP / VPNv4
- [[ospf]] — IGP 의 OSPF-TE
- [[../osi-7-layer/layer-3-network/ipsec-vpn]] — VXLAN / EVPN 비교
- [[routing]] — 라우팅 hub
