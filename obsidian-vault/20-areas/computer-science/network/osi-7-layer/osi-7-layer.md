---
title: "OSI 7 계층 — Hub"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T15:00:00+09:00
tags:
  - network
  - osi
  - layer-model
---

# OSI 7 계층 — Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 깊이 분리 — 각 계층 개별 폴더 / 노트 |

**[[../network|↑ network hub]]** · **[[../../computer-science|↑↑ computer-science]]**

---

## 1. OSI 가 무엇인가

**O**pen **S**ystems **I**nterconnection Reference Model.

ISO (국제표준화기구) 가 1984 년 발표한 **컴퓨터 네트워크 프로토콜의 개념 모델**.
네트워크 통신을 7 개 계층 (Layer) 으로 분리해 각 계층의 책임을 명확히 한다.

각 계층은 보통 **L (Layer)** 로 줄여 부른다:
- L1 = Physical, L2 = Data Link, L3 = Network, ..., L7 = Application

**현대 인터넷의 TCP/IP** 는 이론적 OSI 보다 단순 / 실용 (Simplicity Principle) 을
택해 4 계층 (Network Access / Internet / Transport / Application) 으로 구현되었으나,
설명 / 학습에는 OSI 7 계층이 표준.

---

## 2. 7 계층 — 한눈에

| 계층 | 영문 | 전송 단위 | 대표 장치 | 대표 프로토콜 | 노트 |
| --- | --- | --- | --- | --- | --- |
| L7 | Application | message | L7 Switch, 방화벽, 프록시 | HTTP, FTP, SMTP, DNS, SSH, Telnet | [[layer-7-application/layer-7-application]] |
| L6 | Presentation | data | 코덱 | ASCII, UTF-8, MIME, JPEG, MP3 | [[layer-6-presentation/layer-6-presentation]] |
| L5 | Session | data | L4 Switch | RPC, NetBIOS, SAP | [[layer-5-session/layer-5-session]] |
| L4 | Transport | segment | L4 Switch | TCP, UDP, QUIC, SCTP | [[layer-4-transport/layer-4-transport]] |
| L3 | Network | packet | Router, L3 Switch | IP, ICMP, ARP, IGMP, BGP, OSPF | [[layer-3-network/layer-3-network]] |
| L2 | Data Link | frame | L2 Switch, NIC, AP | Ethernet, Wi-Fi, PPP, HDLC, MAC | [[layer-2-data-link/layer-2-data-link]] |
| L1 | Physical | bit | Cable, Hub, Repeater | RS-232, 10BASE-T, IEEE 802.3 | [[layer-1-physical/layer-1-physical]] |

---

## 3. OSI vs TCP/IP — 매핑

```
OSI 7 계층                  TCP/IP 4 계층
─────────────────────────────────────────
L7 Application
L6 Presentation        →    Application
L5 Session
─────────────────────────────────────────
L4 Transport           →    Transport
─────────────────────────────────────────
L3 Network             →    Internet
─────────────────────────────────────────
L2 Data Link           →    Network Access
L1 Physical                 (Link Layer)
─────────────────────────────────────────
```

| 비교 | OSI | TCP/IP |
| --- | --- | --- |
| 계층 수 | 7 | 4 |
| 출처 | ISO 1984 | DARPA / IETF |
| 성격 | 이론 / 모델 | 실용 / 구현 |
| 표준 단체 | ISO/IEC | IETF |
| 현실 | 학습용 | 실제 인터넷 |

---

## 4. 데이터 캡슐화 (Encapsulation)

송신 시 각 계층은 **헤더 (header) 를 추가** — 수신 시 떼어낸다.

```
[L7] User Data
[L6] User Data
[L5] User Data
[L4] [TCP Header] [User Data]              ← Segment
[L3] [IP Header] [TCP Header] [Data]       ← Packet
[L2] [MAC Header] [IP Hdr] [TCP Hdr] [Data] [FCS]   ← Frame
[L1] bit stream …                          ← Bits
```

수신은 역순:
```
L1 bits → L2 frame → L3 packet → L4 segment → L7 message
```

---

## 5. 각 계층의 주소 / 식별자

| 계층 | 주소 | 예 |
| --- | --- | --- |
| L7 | URL / URI | `https://app.com/users` |
| L4 | Port | 80, 443, 22 |
| L3 | IP | 192.168.1.1, 2001:db8::1 |
| L2 | MAC | 00:1A:2B:3C:4D:5E |
| L1 | 매체 / 회선 | Cat 6, OS2 fiber |

---

## 6. 계층별 단위 / 장치 / 프로토콜 표준

### L1 (Physical)
- 단위: bit
- 장치: 케이블, 안테나, 허브, 리피터, 모뎀, NIC PHY
- 프로토콜: PCM, RS-232/422/485, 10BASE-T, 1000BASE-T
- 표준: IEEE 802.3 (Ethernet PHY), IEEE 802.11 (Wi-Fi PHY)

### L2 (Data Link)
- 단위: frame
- 장치: L2 Switch, Bridge, NIC, 기지국, AP
- 프로토콜: Ethernet, Wi-Fi, PPP, HDLC, ARP, STP, VLAN
- 표준: IEEE 802.x

### L3 (Network)
- 단위: packet
- 장치: Router, L3 Switch
- 프로토콜: IPv4, IPv6, ICMP, ARP, IGMP, NAT, VPN
- 표준: IETF (RFC)

### L4 (Transport)
- 단위: segment (TCP) / datagram (UDP)
- 장치: L4 Switch (NAT, LB)
- 프로토콜: TCP, UDP, QUIC, SCTP
- 표준: IETF

### L5 (Session)
- 단위: data
- 프로토콜: NetBIOS, RPC, SAP, SOCKS
- 실제로는 응용 (L7) 에 흡수되는 경우 많음

### L6 (Presentation)
- 단위: data
- 프로토콜: ASCII, UTF-8, JPEG, MP3, MPEG, MIME, TLS (애매)
- 데이터 표현 / 변환 / 압축 / 암호화

### L7 (Application)
- 단위: message
- 장치: L7 Switch (Application LB, WAF), Proxy
- 프로토콜: HTTP, FTP, SMTP, DNS, SSH, Telnet, RDP
- 표준: IETF, W3C, IEEE

---

## 7. 표준 단체

| 단체 | 영역 |
| --- | --- |
| **IEEE** | L1/L2 — Ethernet, Wi-Fi |
| **3GPP** | L1/L2 — 모바일 (LTE, 5G) |
| **IETF** | L3-L5 — RFC, IP, TCP, HTTP |
| **W3C** | L7 — HTML, CSS, WebSocket |
| **ISO/IEC** | OSI 모델 자체 |
| **ITU-T** | 통신 일반 (X.25, T.30) |

---

## 8. 학습 자료

- **Computer Networking: A Top-Down Approach** (Kurose / Ross)
- **TCP/IP Illustrated Vol.1** (Stevens) — 고전
- **High Performance Browser Networking** (Grigorik) — 무료 https://hpbn.co
- [Cloudflare Learning](https://www.cloudflare.com/learning/)

---

## 9. 관련

- [[../network|↑ network hub]]
- [[../tcp/tcp]] — L4 깊이
- [[../ip/ip]] — L3 깊이
- [[../http/http]] — L7 깊이
- [[../../computer-science|↑↑ computer-science]]
