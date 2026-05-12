---
title: "IP — Internet Protocol (Hub)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T18:55:00+09:00
tags:
  - network
  - ip
  - ipv4
  - ipv6
  - layer-3
---

# IP — Internet Protocol (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | IP hub + 5 세부 노트 |

**[[../network|↑ network hub]]** · **[[../osi-7-layer/layer-3-network/layer-3-network|↑↑ L3 Network]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **전송 단위** | Packet |
| **주소 길이** | IPv4 = 32 bit, IPv6 = 128 bit |
| **신뢰성** | Best-effort (비신뢰) |
| **표준** | RFC 791 (IPv4, 1981), RFC 8200 (IPv6, 2017) |
| **상위 프로토콜** | TCP (6), UDP (17), ICMP (1), GRE (47), ESP (50) |

---

## 1. 한 줄 정의

**서로 다른 네트워크의 호스트들에게 패킷 전달** 을 위한 비신뢰성 / 비연결 프로토콜.
RFC 791 (Postel 1981) — TCP 와 함께 인터넷의 기둥.

---

## 2. IP 의 책임

1. **Logical Addressing** — IP 주소로 호스트 식별
2. **Routing** — 경로 선택
3. **Fragmentation** — MTU 초과 시 단편화 (IPv4 만)
4. **TTL / Hop Limit** — 루프 방지
5. **Encapsulation** — L4 segment 를 L2 에 실음

---

## 3. IPv4 vs IPv6 한눈에

| 측면 | IPv4 | IPv6 |
| --- | --- | --- |
| 주소 길이 | 32 bit (~43억) | 128 bit (3.4×10³⁸) |
| 표기 | `192.168.1.1` | `2001:db8::1` |
| 헤더 | 20-60 byte (가변) | **40 byte (고정)** |
| 헤더 Checksum | 있음 | **없음** (L2 + L4 에 의지) |
| 단편화 주체 | 라우터 + 송신자 | **송신자만** |
| 최소 MTU | 576 (1280 권장) | **1280 (강제)** |
| 주소 부족 | 2011 IANA 소진 | 사실상 무한 |
| ARP | 사용 | **NDP** (ICMPv6 기반) |
| Multicast | 옵션 | 표준 |
| Broadcast | 있음 | **없음** (Multicast 로 대체) |
| 자동 구성 | DHCP 필요 | **SLAAC** + DHCPv6 |
| IPsec | 옵션 | 표준 (실제론 선택) |
| QoS | DSCP/ECN | DSCP/ECN + Flow Label |

자세히 → [[ipv4-vs-ipv6]]

---

## 4. IP 패킷 구조 — 짧게

### IPv4 (20-60 byte)
```
[Ver|IHL|DSCP|Total Length]
[ID|Flags|Fragment Offset]
[TTL|Protocol|Header Checksum]
[Source IP (4)]
[Dest IP (4)]
[Options (가변)]
```

자세히 → [[ipv4-header]]

### IPv6 (40 byte 고정)
```
[Ver|Traffic Class|Flow Label]
[Payload Length|Next Header|Hop Limit]
[Source IPv6 (16)]
[Dest IPv6 (16)]
```

자세히 → [[ipv6-header]]

---

## 5. IP 주소의 분류

### 5.1 IPv4

| 범위 | 분류 |
| --- | --- |
| 0.0.0.0/8 | 예약 (자신) |
| **10.0.0.0/8** | 사설 (RFC 1918) |
| 100.64.0.0/10 | CGNAT (RFC 6598) |
| 127.0.0.0/8 | Loopback |
| 169.254.0.0/16 | Link-local |
| **172.16.0.0/12** | 사설 |
| **192.168.0.0/16** | 사설 |
| 224.0.0.0/4 | Multicast |
| 240.0.0.0/4 | 예약 |
| 255.255.255.255 | Limited Broadcast |

### 5.2 IPv6

| 범위 | 분류 |
| --- | --- |
| `::/128` | 미지정 |
| `::1/128` | Loopback |
| `fe80::/10` | Link-local |
| `fc00::/7` | ULA (사설) |
| `ff00::/8` | Multicast |
| `2000::/3` | Global Unicast |
| `2001:db8::/32` | 문서용 (예시) |

---

## 6. CIDR & Subnetting

자세히 → [[cidr-subnetting]]

```
192.168.1.0/24    ← 24-bit 네트워크 / 8-bit 호스트
2001:db8::/32     ← IPv6 /32 prefix
```

---

## 7. NAT — Network Address Translation

자세히 → [[nat]]

- IPv4 부족 해결의 임시방편
- IPv6 에선 (대체로) 불필요
- 종류: SNAT / DNAT / PAT / Hairpin / CGNAT

---

## 8. 세부 노트 — 이 폴더의 깊이

| 노트 | 영역 |
| --- | --- |
| [[ipv4-header]] | IPv4 헤더 필드 전체 + DSCP/ECN + Options |
| [[ipv6-header]] | IPv6 헤더 + Extension Headers + Flow Label |
| [[cidr-subnetting]] | CIDR / Subnet 계산 / VLSM / Supernetting |
| [[nat]] | SNAT / DNAT / PAT / Hairpinning / CGNAT / NAT64 |
| [[ipv4-vs-ipv6]] | 깊은 비교 / Migration / Dual-stack / Happy Eyeballs |

---

## 9. 면접 / 토픽

1. **IPv4 가 부족한 이유** + 해결 (NAT, IPv6, CGNAT).
2. **사설 IP 범위** (3 가지).
3. **CIDR notation** + **Subnet 계산**.
4. **IPv4 vs IPv6 헤더 차이**.
5. **NAT 의 단점** (P2P, end-to-end).
6. **Link-local vs Global**.
7. **TTL 의 역할 + Traceroute**.
8. **Fragmentation 의 비용**.

---

## 10. 학습 자료

- RFC 791 (IPv4), RFC 8200 (IPv6), RFC 1918 (사설 IP), RFC 4291 (IPv6 주소)
- **TCP/IP Illustrated Vol.1** Ch. 4-7
- **IPv6 Essentials** (Silvia Hagen)
- ARIN / RIPE / APNIC — Regional Internet Registry

---

## 11. 관련

- [[../osi-7-layer/layer-3-network/layer-3-network]] — L3 hub
- [[../routing/routing|↗ 라우팅]] — IP 위의 라우팅 프로토콜
- [[../osi-7-layer/layer-3-network/arp]] — ARP (IPv4 IP→MAC)
- [[../osi-7-layer/layer-3-network/icmp]] — ICMP
- [[../osi-7-layer/layer-3-network/fragmentation-mtu]] — 단편화 / MTU
