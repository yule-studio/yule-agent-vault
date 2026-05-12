---
title: "L3 — 네트워크 계층 (Network Layer)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T16:25:00+09:00
tags:
  - network
  - osi
  - layer-3
  - ip
---

# L3 — 네트워크 계층 (Network Layer)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | L3 hub + 세부 노트 인덱스 |

**[[../osi-7-layer|↑ OSI 7 계층]]** · **[[../../network|↑↑ network hub]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **전송 단위** | **Packet (패킷)** |
| **주소 / 식별자** | **IP 주소** (IPv4 32 bit, IPv6 128 bit) |
| **대표 장치** | Router, L3 Switch, Firewall, VPN concentrator |
| **대표 프로토콜** | **IPv4, IPv6, ICMP, ICMPv6, ARP, NDP, IGMP, IPsec**, RIP/OSPF/BGP (라우팅) |
| **상위 계층** | L4 Transport |
| **하위 계층** | L2 Data Link |

---

## 1. 한 줄 정의

**서로 다른 네트워크 간 패킷을 목적지까지 전달** 하는 계층. **IP 주소** 기반 라우팅,
**Fragmentation**, **TTL** 등 인터넷의 본질이 여기 있다.

L2 가 "인접 노드" 라면 L3 는 "끝-끝 (end-to-end) 망 도달".

---

## 2. L3 의 4 가지 책임

### 2.1 Logical Addressing (논리 주소)
- IP 주소 — 위치 / 그룹을 표현
- L2 MAC 은 출고 때 고정 / 위치 모름. IP 는 토폴로지 반영.
- 자세히 → [[../../ip/ip|↗ IP hub]]

### 2.2 Routing (라우팅)
- 패킷을 어느 경로로 보낼 것인가
- 라우팅 테이블 + 동적 프로토콜 (RIP, OSPF, BGP)
- 자세히 → [[../../routing/routing|↗ 라우팅 hub]]

### 2.3 Fragmentation / Reassembly
- 큰 패킷을 작은 MTU 에 맞게 분할
- 수신 측에서 재조립 (IPv4) / 송신 측에서 PMTUD (IPv6)
- 자세히 → [[fragmentation-mtu]]

### 2.4 Error / Diagnostic (ICMP)
- 라우터가 문제 알림 (Destination Unreachable, TTL Exceeded)
- ping / traceroute 의 기반
- 자세히 → [[icmp]]

---

## 3. L2 vs L3 핵심 비교

| 측면 | L2 (Data Link) | L3 (Network) |
| --- | --- | --- |
| 주소 | MAC (48 bit) | IP (32/128 bit) |
| 범위 | 인접 노드 (LAN) | 망 전체 (인터넷) |
| 라우팅 | 없음 (Switching) | 라우팅 테이블 + 프로토콜 |
| TTL | 없음 (STP 필요) | TTL (16-bit hop limit) |
| 단편화 | 없음 | IPv4 (라우터), IPv6 (송신) |
| 신뢰성 | CRC + ACK | 비신뢰 (TCP 가 보장) |
| 표준 단체 | IEEE | IETF (RFC) |

---

## 4. 같은 LAN vs 다른 LAN 통신

### 4.1 같은 서브넷

```
PC1 (192.168.1.5) → PC2 (192.168.1.10)
   ↓ ARP "192.168.1.10 의 MAC?"
   ↓
   ARP Reply: MAC = AA:BB:CC:DD:EE:FF
   ↓
   L2 프레임: Dest MAC = AA:BB... , IP = 192.168.1.10
```

### 4.2 다른 서브넷 (Default Gateway 통해)

```
PC1 (192.168.1.5, GW=192.168.1.1) → 8.8.8.8
   ↓ "8.8.8.8 은 내 서브넷 아님 → GW 로"
   ↓ ARP "192.168.1.1 의 MAC?"  → Router MAC
   ↓
   L2 프레임: Dest MAC = Router , IP = 8.8.8.8
   ↓ Router 가 받음
   ↓ 라우팅 테이블 검색
   ↓ 다음 hop 의 L2 프레임 재생성 (src/dst MAC 바뀜)
   ↓ ... 최종 도달 ...
```

→ **IP 는 그대로, MAC 은 매 hop 마다 바뀜**.

---

## 5. IP 패킷 구조 (IPv4 / IPv6) — 인덱스

자세히 → [[../../ip/ipv4-header]], [[../../ip/ipv6-header]]

### IPv4 헤더 (20-60 byte)
```
[Ver][IHL][DSCP/ECN][Total Length]
[ID][Flags][Fragment Offset]
[TTL][Protocol][Header Checksum]
[Source IP (4)]
[Destination IP (4)]
[Options (가변)]
```

### IPv6 헤더 (40 byte 고정)
```
[Ver][TC][Flow Label]
[Payload Length][Next Header][Hop Limit]
[Source IPv6 (16)]
[Destination IPv6 (16)]
```

IPv6 는 헤더 단순화 — fragmentation / checksum 제거 (L2 + L4 가 처리).

---

## 6. 주요 세부 노트

| 노트 | 영역 |
| --- | --- |
| [[arp]] | ARP / Gratuitous ARP / RARP / Proxy ARP / NDP |
| [[icmp]] | ICMP / ICMPv6 / ping / traceroute |
| [[fragmentation-mtu]] | IPv4 단편화 / PMTUD / IPv6 차이 |
| [[ip-routing-basics]] | 라우팅 테이블 / Longest Prefix Match / Default GW |
| [[multicast-igmp]] | IPv4 Multicast / IGMP / IPv6 MLD |
| [[ipsec-vpn]] | IPsec / GRE / VXLAN / VPN |

또한 별도 폴더로 깊이:
- [[../../ip/ip|↗ IP 폴더]] — IPv4 / IPv6 / Subnet / CIDR / NAT / 헤더 분석
- [[../../routing/routing|↗ 라우팅 폴더]] — RIP / OSPF / BGP / MPLS

---

## 7. L3 의 함정

### 함정 1 — IP 가 신뢰 제공 가정
IP 는 **best-effort** — 손실 / 중복 / 순서 깨짐 가능. TCP 가 보장.

### 함정 2 — Subnet Mask 잘못
같은 LAN 인데 다른 서브넷 → ARP X, 통신 X.

### 함정 3 — Default Gateway 미설정
같은 LAN 만 통신 가능, 외부 X.

### 함정 4 — Fragmentation 의 성능 영향
재조립 비용, 일부 손실 시 전체 폐기. PMTUD 권장.

### 함정 5 — ICMP 차단
방화벽이 ICMP 차단 → PMTUD 실패 → "ICMP black hole".

### 함정 6 — TTL = 0
라우팅 루프 시 패킷 무한 순환 방지. 256 hop 한계.

### 함정 7 — IPv4 / IPv6 dual-stack 의 복잡성
양 stack 운영 비용. v6 우선 (Happy Eyeballs) RFC 8305.

---

## 8. 면접 / 토픽

1. **L2 vs L3 차이** — MAC vs IP, Switch vs Router.
2. **같은 LAN 통신 vs 다른 LAN 통신** — ARP / Default Gateway.
3. **IP 패킷 구조** — IPv4 헤더 필드.
4. **Fragmentation** — 발생 / 위험.
5. **TTL 의 역할**.
6. **ICMP / traceroute 원리**.
7. **IPv4 vs IPv6** — 주소 / 헤더 / 단편화 차이.

---

## 9. 학습 자료

- **TCP/IP Illustrated Vol.1** (Stevens) — Ch. 4-7
- **Computer Networking: Top-Down** (Kurose) — Ch. 4-5
- RFC 791 (IPv4), RFC 8200 (IPv6)
- RFC 1122 (Host requirements)

---

## 10. 관련

- [[../layer-2-data-link/layer-2-data-link]] — 하위
- [[../layer-4-transport/layer-4-transport]] — 상위
- [[../../ip/ip|↗ IP 깊이 폴더]]
- [[../../routing/routing|↗ 라우팅 폴더]]
- [[../osi-7-layer]] — OSI hub
