---
title: "Multicast & IGMP / MLD"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T16:50:00+09:00
tags:
  - network
  - layer-3
  - multicast
  - igmp
---

# Multicast & IGMP / MLD

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Multicast 주소 / IGMP / IGMP Snooping / MLD / PIM 인덱스 |

**[[layer-3-network|↑ L3 Network]]** · **[[../osi-7-layer|↑↑ OSI 7]]**

---

## 1. 전달 방식 4 가지

| 방식 | 송신 | 수신 | 예 |
| --- | --- | --- | --- |
| **Unicast** | 1 | 1 | HTTP 일반 |
| **Broadcast** | 1 | 모두 (LAN) | ARP |
| **Multicast** | 1 | 그룹 가입자만 | IPTV, OSPF, mDNS |
| **Anycast** | 1 | 가장 가까운 1 | DNS Root, CDN |

---

## 2. Multicast — 1:N

### 2.1 동작 원리

```
Source → Multicast Group (224.1.1.1)
   ↓
Router 가 그룹 가입한 호스트가 있는 LAN 으로만 복제
   ↓
LAN 안 Switch 가 그룹 가입자에게만 전달 (IGMP Snooping)
```

→ 단일 송신, 효율적 복제.

### 2.2 장점
- 동시 다수에게 효율적 (vs N 개 unicast)
- 대역폭 절약
- 송신자 부담 ↓

### 2.3 단점 / 한계
- 인터넷에선 잘 안 됨 (BGP / NAT 제약)
- 라우터 / Switch 의 구현 복잡
- 신뢰성 X (UDP 기반)

### 2.4 사용
- IPTV / 비디오 스트리밍 (사내)
- 주식 시세 (HFT)
- 라우팅 프로토콜 (OSPF, RIPv2)
- mDNS / Bonjour / SSDP / UPnP
- LAN 내 발견 (Sonos, Chromecast)

---

## 3. Multicast 주소

### 3.1 IPv4 — Class D

```
224.0.0.0 - 239.255.255.255
첫 4 bit = 1110
```

| 범위 | 용도 |
| --- | --- |
| **224.0.0.0/24** | Link-local — 같은 LAN 만 (TTL=1) |
| 224.0.0.1 | All Hosts |
| 224.0.0.2 | All Routers |
| 224.0.0.5 | OSPF |
| 224.0.0.6 | OSPF DR |
| 224.0.0.9 | RIPv2 |
| 224.0.0.13 | PIM |
| 224.0.0.18 | VRRP |
| 224.0.0.22 | IGMPv3 |
| 224.0.0.251 | **mDNS (Bonjour)** |
| 224.0.0.252 | LLMNR |
| 239.0.0.0/8 | **Administratively Scoped** (Private) |
| 232.0.0.0/8 | **SSM** (Source-Specific Multicast) |

### 3.2 IPv6

| 범위 | 용도 |
| --- | --- |
| `FF00::/8` | Multicast |
| `FF02::1` | All nodes (link) |
| `FF02::2` | All routers (link) |
| `FF02::5` | OSPFv3 |
| `FF02::FB` | mDNS |

#### Scope (IPv6)
- `FF01::` — Interface-local
- `FF02::` — Link-local
- `FF05::` — Site-local
- `FF08::` — Organization-local
- `FF0E::` — Global

### 3.3 IP → MAC Multicast 매핑

#### IPv4
- IPv4 multicast 의 하위 23 bit → MAC 의 하위 23 bit
- MAC 형식: `01:00:5E:xx:xx:xx`

예: `224.1.2.3` → `01:00:5E:01:02:03`
예: `239.255.255.250` (SSDP) → `01:00:5E:7F:FF:FA`

→ 32 → 23 bit 매핑이라 **32 IP 가 같은 MAC** 일 수 있음 (overlap).

#### IPv6
- 형식: `33:33:xx:xx:xx:xx`
- 마지막 32 bit 가 IPv6 mcast 의 마지막 32 bit

---

## 4. IGMP — Internet Group Management Protocol

LAN 의 호스트가 라우터에게 **"이 multicast 그룹 가입 / 탈퇴"** 알림. RFC 3376 (IGMPv3).

### 4.1 IGMP 버전

| 버전 | 주요 기능 |
| --- | --- |
| **v1** (RFC 1112) | 단순 Join/Query |
| **v2** (RFC 2236) | Leave 메시지 — 빠른 탈퇴 |
| **v3** (RFC 3376) | **SSM 지원** — 특정 소스만 |

### 4.2 IGMP 메시지

| Type | 의미 |
| --- | --- |
| **Membership Query** | 라우터: "누가 그룹 가입했나?" |
| **Membership Report** | 호스트: "나 가입 중" |
| **Leave Group** | 호스트: "나 탈퇴" (v2+) |

### 4.3 IGMP 동작

```
1. 라우터: Membership Query (224.0.0.1, 주기적)
2. 호스트 (가입자): Membership Report (각 그룹)
3. 다른 호스트는 보고 가만히 (Report Suppression — v3 제거)
4. 호스트 탈퇴: Leave Group → 라우터 즉시 forwarding 중단
```

### 4.4 IGMP Snooping

L2 Switch 가 IGMP 메시지 엿보기 → 멤버 포트 학습 → multicast 를 멤버 포트에만.

```
IGMP Snooping OFF:  multicast → 모든 포트 broadcast (낭비)
IGMP Snooping ON:   multicast → 가입 포트만
```

→ 모던 enterprise Switch 의 표준 기능.

### 4.5 IGMP Querier
- IGMP Snooping 이 동작하려면 Querier 필요
- 일반적으로 router 가 함
- Router 없는 LAN 에서는 Switch 가 Querier 활성화

---

## 5. MLD — Multicast Listener Discovery (IPv6)

IPv6 의 IGMP. ICMPv6 메시지로:

| Type | 의미 |
| --- | --- |
| 130 | MLD Query |
| 131 | MLD Report (v1) |
| 132 | MLD Done (Leave) |
| 143 | MLDv2 Report |

기능은 IGMP 와 동일.

---

## 6. 라우터 간 — PIM (Protocol Independent Multicast)

LAN 간 multicast 전달. 자세히 → [[../../routing/pim]] (예정)

### 6.1 종류
- **PIM-DM** (Dense Mode) — Flood-and-prune
- **PIM-SM** (Sparse Mode) — RP (Rendezvous Point) 기반 — 표준
- **PIM-SSM** (Source-Specific Multicast) — (S,G) 직접

### 6.2 PIM 의 의의
- 모든 라우팅 프로토콜과 무관 ("Independent")
- 기본 unicast 라우팅 테이블 사용 (RPF — Reverse Path Forwarding)

---

## 7. SSM (Source-Specific Multicast, RFC 3569)

- (Source, Group) tuple 로 구독
- 232.0.0.0/8 범위
- 인터넷 multicast 의 현실적 모델 (다른 모델 — ASM — 은 안 통함)

---

## 8. Anycast — 가장 가까운 1

같은 IP 를 여러 장소가 광고 → 라우팅이 가장 가까운 곳으로.

### 8.1 사용
- **DNS Root server** — 13 개 명목, 실제 수백 개 anycast
- **CDN** — Cloudflare 1.1.1.1, Google 8.8.8.8
- **DDoS 완화** — 부하 분산

### 8.2 동작
- 같은 prefix 를 여러 AS 가 BGP 광고
- BGP 가 가장 짧은 경로 선택
- TCP 에선 비대칭 경로 위험 — UDP 가 보통

---

## 9. mDNS / Bonjour

- 224.0.0.251 (`FF02::FB`) UDP 5353
- LAN 안 디바이스 자동 발견 (.local 도메인)
- DNS 와 같은 형식, broadcast 대신 multicast
- Apple AirDrop / AirPrint / HomeKit
- Avahi (Linux)

---

## 10. SSDP (Simple Service Discovery Protocol)

- 239.255.255.250 UDP 1900
- UPnP 의 발견 부분
- Smart TV / 라우터 / Sonos

---

## 11. 실용 — Multicast 디버깅

```bash
# Linux — multicast 가입 확인
ip maddress show
netstat -gn

# 그룹 가입 (테스트)
python3 -c "
import socket, struct
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM, socket.IPPROTO_UDP)
s.bind(('', 5000))
mreq = struct.pack('4sl', socket.inet_aton('224.1.1.1'), socket.INADDR_ANY)
s.setsockopt(socket.IPPROTO_IP, socket.IP_ADD_MEMBERSHIP, mreq)
while True:
    print(s.recv(1024))
"

# 송신
python3 -c "
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM, socket.IPPROTO_UDP)
s.sendto(b'hello', ('224.1.1.1', 5000))
"
```

```bash
# tcpdump 로 multicast 캡처
sudo tcpdump -i en0 -nn multicast
sudo tcpdump -i en0 -nn 'host 224.0.0.0/4'
```

---

## 12. 함정

### 함정 1 — Internet multicast 가정
대부분 ISP 가 multicast 차단. LAN / 사내만.

### 함정 2 — IGMP Snooping 없는 Switch
multicast 가 broadcast 처럼 → 모든 포트로. CCTV / IPTV 환경 마비.

### 함정 3 — IGMP Querier 없음
Snooping 동작 X → 같은 결과.

### 함정 4 — TTL=1 의 mcast
Link-local — 라우터 통과 X. 의도된 경우 (mDNS).

### 함정 5 — MAC overlap
32 IPv4 가 같은 MAC → 의도치 않은 호스트가 받음.

### 함정 6 — NAT / 방화벽
대부분 multicast 통과 X.

### 함정 7 — VLAN 경계
multicast 는 VLAN 안에서만 — 다른 VLAN 보내려면 라우터 + PIM.

---

## 13. 학습 자료

- RFC 1112 (Host extensions for multicast), RFC 3376 (IGMPv3)
- RFC 4604 (PIM-SM), RFC 3569 (SSM)
- **Multicast Networking and Applications** (Wittmann)
- Cisco "IP Multicast" 백서

---

## 14. 관련

- [[icmp]] — ICMPv6 (MLD 의 기반)
- [[arp]] — Broadcast 와 비교
- [[../../routing/routing|↗ 라우팅 폴더]] (PIM)
- [[layer-3-network]] — 상위
