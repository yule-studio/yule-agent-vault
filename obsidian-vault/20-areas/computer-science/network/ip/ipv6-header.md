---
title: "IPv6 헤더 & Extension Headers"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T19:05:00+09:00
tags:
  - network
  - ipv6
  - ip-header
---

# IPv6 헤더 & Extension Headers

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 헤더 / Extension Headers / Flow Label / 주소 표기 |

**[[ip|↑ IP]]** · **[[../network|↑↑ network hub]]**

---

## 1. 기본 헤더 (40 byte 고정)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version| Traffic Class |           Flow Label                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Payload Length        |  Next Header  |   Hop Limit   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                                                               +
|                                                               |
+                         Source Address                        +
|                                                               |
+                                                               +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                                                               +
|                                                               |
+                      Destination Address                      +
|                                                               |
+                                                               +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

→ **40 byte 고정** (IPv4 의 20-60 가변과 다름).

---

## 2. 필드별 상세

### 2.1 Version (4 bit)
`6`.

### 2.2 Traffic Class (8 bit)
- IPv4 의 DSCP+ECN 와 동일 의미
- 6 bit DSCP + 2 bit ECN

### 2.3 Flow Label (20 bit)

**IPv4 에 없는 새 필드** — 같은 흐름 (flow) 식별:
- Src IP, Dst IP, Flow Label = 한 흐름
- 라우터가 fast path 처리 (헤더 전부 안 봄)
- QoS, MPLS-like
- 실제 활용은 제한적 — 라우터마다 다름

### 2.4 Payload Length (16 bit)
- 헤더 제외 페이로드 + Extension Headers
- 최대 **65535 byte**
- "Jumbogram" (>64KB) 옵션 → Extension Header 로

### 2.5 Next Header (8 bit)

다음 헤더의 종류:
- L4 직접 (Protocol Number 와 같음): TCP=6, UDP=17, ICMPv6=58
- 또는 Extension Header (0, 43, 44, 50, 51, 60 등)

### 2.6 Hop Limit (8 bit)
- IPv4 의 TTL 과 동일
- 매 hop -1 → 0 폐기 + ICMPv6 Type 3

### 2.7 Source Address (128 bit)
출발 IPv6.

### 2.8 Destination Address (128 bit)
목적 IPv6.

---

## 3. IPv6 헤더의 단순화

IPv4 에서 **제거된** 필드:
- **IHL** — 헤더 고정 40
- **Identification / Flags / Fragment Offset** — 라우터 단편화 X
- **Header Checksum** — L2 CRC + L4 가 충분
- **Options** — Extension Headers 로

→ 라우터가 매 hop 마다 처리할 게 줄어 빠름.

---

## 4. Extension Headers — 옵션의 진화

```
[IPv6 Header] → [Hop-by-Hop EH] → [Routing EH] → [Fragment EH] → [Payload (L4)]
   Next=0          Next=43           Next=44         Next=6 (TCP)
```

체인 형태 — Next Header 가 다음 EH 또는 L4 가리킴.

### 4.1 종류

| Next | 이름 | 용도 |
| --- | --- | --- |
| **0** | **Hop-by-Hop Options** | 모든 라우터가 처리 (예: Jumbogram, Router Alert) |
| **43** | **Routing** | 명시 경로 (Type 0 deprecated, Type 2 모바일, Type 4 SRH) |
| **44** | **Fragment** | 송신자의 단편화 |
| **50** | **ESP** (IPsec) | 암호화 |
| **51** | **AH** (IPsec) | 인증 |
| **59** | **No Next Header** | 페이로드 없음 |
| **60** | **Destination Options** | 목적지만 처리 (예: Mobile IPv6) |
| **135** | **Mobility** | Mobile IPv6 |

### 4.2 처리 순서 (송신자 권장)
1. Hop-by-Hop (있으면 반드시 처음)
2. Destination (라우팅 헤더 있을 때 — 중간 경유지)
3. Routing
4. Fragment
5. AH
6. ESP
7. Destination (최종 목적지)
8. L4 (TCP/UDP/...)

### 4.3 SRH — Segment Routing Header (RFC 8754)

- 패킷에 경로 명시 (segment list)
- SRv6 의 기반 — SDN 의 새 표준
- MPLS 대체 후보

---

## 5. IPv6 주소 표기

### 5.1 16 진수 8 그룹 × 16 bit

```
2001:0db8:85a3:0000:0000:8a2e:0370:7334
```

### 5.2 축약 규칙

```
1. 각 그룹의 앞쪽 0 제거:
   2001:db8:85a3:0:0:8a2e:370:7334

2. 연속 0 그룹은 `::` 로 한 번만:
   2001:db8:85a3::8a2e:370:7334

3. 두 번 사용 금지:
   2001::1::1   ← X
```

### 5.3 IPv4-mapped IPv6

```
::ffff:192.0.2.1   = ::ffff:c000:0201
```

IPv4 호환성 — dual-stack 소켓에서 IPv4 연결 표현.

### 5.4 IPv4-translated (RFC 6052)

```
64:ff9b::192.0.2.1   ← NAT64 표준 prefix
```

---

## 6. 주소 종류 (Unicast / Multicast / Anycast)

### 6.1 Unicast
- 단일 인터페이스
- **Global** — `2000::/3`
- **Link-local** — `fe80::/10` (각 인터페이스 자동 부여)
- **ULA** (Unique Local Address) — `fc00::/7` (사설망)
- **Loopback** — `::1/128`
- **Unspecified** — `::/128`

### 6.2 Multicast
- `ff00::/8`
- Scope (4 bit):
  - 1: Interface-local
  - 2: Link-local
  - 5: Site-local
  - 8: Organization
  - e: Global

| 주소 | 의미 |
| --- | --- |
| `ff02::1` | All Nodes |
| `ff02::2` | All Routers |
| `ff02::5` | OSPFv3 |
| `ff02::fb` | mDNS |
| `ff02::1:ffXX:XXXX` | Solicited-Node (NDP) |

### 6.3 Anycast
- Unicast 와 같은 주소 공간 — 라우팅이 가장 가까운 곳으로
- DNS root, Cloudflare 1.1.1.1 (IPv4 도 같은 개념)

### 6.4 No Broadcast
- IPv6 는 **Broadcast 없음**
- All-Nodes Multicast (`ff02::1`) 로 대체

---

## 7. SLAAC — Stateless Address Autoconfiguration (RFC 4862)

```
1. 호스트 부팅 → 자기 Link-local (fe80::eui64)
2. NS (Neighbor Solicitation) 로 중복 감지 (DAD)
3. RS (Router Solicitation) → 라우터에게 "Prefix 알려줘"
4. RA (Router Advertisement) → "Prefix=2001:db8::/64"
5. 호스트가 IPv6 = Prefix + EUI-64 (또는 random)
```

### EUI-64
- MAC 48 bit → 64 bit (중간에 0xFFFE 삽입 + bit 7 반전)
- MAC 노출 프라이버시 X → **Privacy Extensions** (RFC 8981) 로 random

### DHCPv6
- 일부 환경 — DNS 서버 등 추가 정보
- M flag (RA 의 Managed) → DHCPv6 사용

---

## 8. NDP — Neighbor Discovery Protocol

IPv6 의 ARP. ICMPv6 메시지 (Type 133-137).

자세히 → [[../osi-7-layer/layer-3-network/arp#11 NDP]]

---

## 9. 단편화 (IPv6)

- **라우터는 단편화 X** — 못 보내면 폐기 + ICMPv6 Type 2 (Packet Too Big)
- **송신자가 PMTUD** 필수
- 송신자가 Fragment Extension Header 추가
- 최소 MTU = **1280 byte** 보장

자세히 → [[../osi-7-layer/layer-3-network/fragmentation-mtu#8 IPv6 와 단편화]]

---

## 10. IPv6 의 정책

### Privacy
- **Privacy Extensions** (RFC 8981) — 임시 주소
- 출력은 random, 받기는 EUI-64
- Windows / macOS / iOS 모두 활성

### Mobile IPv6 (RFC 6275)
- Home Address + Care-of Address
- 이동해도 같은 주소 유지
- 거의 사용 X — VPN 으로 대체

---

## 11. 도입 현황 (2025)

- Google 측정 IPv6 트래픽: **약 45%**
- 미국 / 인도 / 프랑스 / 독일 / 일본: 60%+
- 한국: ~30%
- 모바일 (LTE/5G): 80%+

→ 점진 dual-stack → IPv6-only 모바일망.

---

## 12. 함정

### 함정 1 — `::1` ↔ `127.0.0.1` 혼동
서버가 `::1` listen 시 IPv4 클라이언트 접속 X (별도 listen 필요).

### 함정 2 — Link-local 주소만으로 통신
인터페이스 명시 필요: `fe80::1%eth0`.

### 함정 3 — Privacy Extensions 의 IP 자주 변함
방화벽 IP 차단 / 로그 정합성 어려움.

### 함정 4 — IPv6 의 NAT 가정
원칙적으로 X — 끝-끝 통신. 일부 NAT66 / NPTv6 (RFC 6296) 존재.

### 함정 5 — ICMPv6 차단
NDP 안 동작 → 통신 자체 X. IPv4 의 ICMP 와 다르게 절대 차단 X.

### 함정 6 — Extension Headers 의 미들박스 호환성
방화벽 / DPI 가 모르는 EH 폐기 — 일부 deployment 어려움.

### 함정 7 — IPv4-mapped IPv6 의 보안 우회
`::ffff:127.0.0.1` 이 localhost 검사 우회 — 입력 검증 시 주의.

---

## 13. tcpdump / 도구

```bash
sudo tcpdump -i en0 -nn -v ip6

# IPv6 ping
ping -6 google.com
ping6 google.com

# 주소 확인
ip -6 addr
ifconfig en0 | grep inet6

# 경로
ip -6 route
traceroute6 google.com
```

---

## 14. 학습 자료

- RFC 8200 (IPv6), RFC 4291 (IPv6 주소), RFC 4861 (NDP), RFC 4862 (SLAAC)
- RFC 8981 (Privacy Extensions), RFC 8754 (SRH)
- **IPv6 Essentials** (Silvia Hagen)
- Google IPv6 통계 — https://www.google.com/intl/en/ipv6/statistics.html

---

## 15. 관련

- [[ip]] — IP hub
- [[ipv4-header]] — IPv4 헤더 비교
- [[ipv4-vs-ipv6]] — 깊은 비교
- [[nat]] — NAT64 / NPTv6
- [[../osi-7-layer/layer-3-network/arp]] — NDP
