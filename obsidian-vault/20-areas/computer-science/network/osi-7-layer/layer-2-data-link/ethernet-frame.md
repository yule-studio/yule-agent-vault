---
title: "Ethernet 프레임 (Frame Structure)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T15:40:00+09:00
tags:
  - network
  - layer-2
  - ethernet
  - frame
---

# Ethernet 프레임 (Frame Structure)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Ethernet II / 802.3 / VLAN / Jumbo / 프레이밍 기법 |

**[[layer-2-data-link|↑ L2 Data Link]]** · **[[../osi-7-layer|↑↑ OSI 7]]**

---

## 1. Ethernet 의 역사

| 연도 | 사건 |
| --- | --- |
| 1973 | Robert Metcalfe — Xerox PARC 메모 (Ethernet 발명) |
| 1980 | DEC + Intel + Xerox — Ethernet II 표준 (DIX) |
| 1983 | IEEE 802.3 — 공식 표준 |
| 1985 | 10BASE5 (Thicknet) |
| 1990 | 10BASE-T (UTP) — 가정 / 사무실 보급 |
| 1995 | 100BASE-TX (Fast Ethernet) |
| 1999 | 1000BASE-T (Gigabit) |
| 2002 | 10GBASE — 데이터센터 |
| 2010 | 40 GbE / 100 GbE |
| 2017 | 400 GbE (IEEE 802.3bs) |
| 2022 | 800 GbE 데이터센터 |
| 2024+ | 1.6 TbE 표준 작업 중 |

핵심 통찰: 1973 의 기본 프레임 구조가 50 년간 유지 (호환성). 속도만 100억 배 증가.

---

## 2. Ethernet II (DIX) 프레임 구조

```
┌──────────────┬──────────┬──────────┬───────┬───────────────┬──────┐
│  Preamble    │  SFD     │ Dest MAC │ Src MAC│ EtherType    │ Data │ FCS
│  7 byte      │  1 byte  │  6 byte  │ 6 byte │ 2 byte       │ 46-1500│ 4
│ (10101010×7) │(10101011)│          │        │ (e.g. 0x0800)│       │
└──────────────┴──────────┴──────────┴───────┴───────────────┴──────┘
```

### 2.1 Preamble (7 byte)
- `10101010` × 7
- 수신기 클록 동기화

### 2.2 SFD (Start Frame Delimiter, 1 byte)
- `10101011`
- 프레임 시작 표시

### 2.3 Destination MAC (6 byte)
- 목적지 MAC 주소
- Broadcast: `FF:FF:FF:FF:FF:FF`
- Multicast: 첫 옥텟의 LSB = 1

### 2.4 Source MAC (6 byte)
- 출발지 MAC 주소

### 2.5 EtherType (2 byte)
상위 계층 프로토콜 식별:

| 값 | 프로토콜 |
| --- | --- |
| **0x0800** | IPv4 |
| **0x86DD** | IPv6 |
| **0x0806** | ARP |
| **0x8035** | RARP |
| **0x8100** | VLAN tag (802.1Q) |
| **0x8847** | MPLS unicast |
| **0x8848** | MPLS multicast |
| **0x88CC** | LLDP |
| **0x88E5** | MACsec (802.1AE) |
| **0x88F7** | PTP (802.1AS) |

### 2.6 Data / Payload (46 - 1500 byte)
- 최소 46 byte — 그 미만은 padding
- 최대 1500 byte (표준 MTU)
- IP 패킷 등 상위 계층 데이터

### 2.7 FCS (Frame Check Sequence, 4 byte)
- CRC-32 (다항식 0x04C11DB7)
- 프레임 전체에 대한 무결성 검증

자세히 → [[error-detection-crc]]

### 2.8 IFG (Inter-Frame Gap)
- 프레임 간 최소 간격
- 12 byte 시간 (10 Mbps 에서 9.6 μs)
- 매체 회복 + 수신기 처리 시간

---

## 3. IEEE 802.3 vs Ethernet II 차이

```
Ethernet II:
... [EtherType=0x0800] [IP Payload] ...

IEEE 802.3:
... [Length=46-1500] [LLC 3 byte] [SNAP 5 byte] [Payload] ...
```

- EtherType 자리가 **Length** 로 — 1500 이하 값은 Length, 1536 (0x0600) 이상은 EtherType
- LLC + SNAP 으로 상위 프로토콜 식별 (오버헤드 8 byte)

**현실**: 99% 이상이 **Ethernet II** 사용. 802.3 LLC/SNAP 은 거의 사장.

---

## 4. VLAN 태그된 프레임 (802.1Q)

```
┌─────────┬─────────┬─────────────┬───────────────┬──────┐
│Dest MAC │Src MAC  │ 802.1Q Tag  │ EtherType     │ Data │ FCS
│  6      │  6      │  4 byte     │  2 byte       │      │
└─────────┴─────────┴─────────────┴───────────────┴──────┘

802.1Q Tag (4 byte):
┌──────────────┬──────┬───┬────────┐
│ TPID 0x8100  │ PCP  │DEI│ VID    │
│   16 bit     │  3   │ 1 │ 12 bit │
└──────────────┴──────┴───┴────────┘
```

- **TPID** = 0x8100 — VLAN 표시
- **PCP** (Priority) — QoS 우선순위 (0-7)
- **DEI** (Drop Eligible) — 혼잡 시 폐기 가능
- **VID** — VLAN ID (0-4095, 0/4095 예약)

802.1Q-in-Q (QinQ) 는 2 개 tag — 통신사가 고객 VLAN 위에 자신의 VLAN.

자세히 → [[vlan-stp-bonding]]

---

## 5. Jumbo Frame

표준 MTU 1500 → **Jumbo** 9000 byte.

### 장점
- 패킷당 헤더 오버헤드 ↓
- CPU 부담 ↓ (적은 인터럽트)
- 10G+ 환경에서 효율 ↑

### 단점
- 표준 외 — Internet 끝-끝 전달 X
- MTU mismatch 시 fragmentation / drop
- 모든 장비가 지원해야 함

### 사용
- 데이터센터 내부 (NAS, vMotion, NFS)
- iSCSI / FCoE
- 클러스터 통신

### Baby Giant
- 1518 + VLAN tag + MPLS = 1538 ~ 1554
- 일부 장비 호환성 문제

---

## 6. MTU vs MSS vs Frame Size

| 용어 | 의미 | 값 (Ethernet) |
| --- | --- | --- |
| **Frame Size** | 전체 L2 프레임 | 14 (header) + 1500 + 4 (FCS) = 1518 |
| **MTU** (Maximum Transmission Unit) | L3 payload 최대 | 1500 |
| **MSS** (Maximum Segment Size) | TCP payload 최대 | 1500 - 20 (IP) - 20 (TCP) = 1460 |

PPPoE — MTU 1492 (8 byte header).
IPsec VPN — MTU 더 작음 (캡슐화 오버헤드).

### Path MTU Discovery (PMTUD)
- 경로 중 가장 작은 MTU 찾기
- ICMP "Fragmentation needed" 메시지
- ICMP 차단 시 PMTUD 실패 → "ICMP black hole"

---

## 7. 프레이밍 (Framing) 기법

L1 의 비트 흐름에서 프레임 경계 구분하는 4 가지 방식:

### 7.1 문자 카운트 (Character Count)
헤더에 길이 명시.

```
[Length=5][a][b][c][d][e]
```

단점: 길이 필드 손상 시 회복 X.

### 7.2 플래그 + Byte Stuffing

- 시작 / 끝 플래그 `0x7E` (또는 `0x7F`)
- 데이터 안 `0x7E` 는 escape (`0x7D 0x5E`)
- PPP 사용

### 7.3 플래그 + Bit Stuffing

- 플래그 `01111110` (6 개 1)
- 데이터에 5 개 연속 1 후 0 강제 삽입 → 플래그와 구분
- HDLC, USB 사용

```
원본:  011111111 → 송신: 0111110111
수신:  0111110111 → 복원: 011111111 (5 개 1 후 0 제거)
```

### 7.4 코딩 위반 (Physical Layer Coding Violation)

- Manchester 의 불가능 패턴 (전이 없음)
- 4B/5B 의 J/K control code
- Ethernet 의 Preamble + SFD

---

## 8. Ethernet 종류 — PHY 표준

| 표준 | 속도 | 매체 | 거리 |
| --- | --- | --- | --- |
| **10BASE5** (Thicknet) | 10 Mbps | RG-8 동축 | 500m |
| **10BASE2** (Thinnet) | 10 Mbps | RG-58 동축 | 185m |
| **10BASE-T** | 10 Mbps | UTP Cat 3+ | 100m |
| **10BASE-FL** | 10 Mbps | MMF | 2 km |
| **100BASE-TX** (Fast) | 100 Mbps | UTP Cat 5 | 100m |
| **100BASE-FX** | 100 Mbps | MMF | 2 km |
| **1000BASE-T** (Gig) | 1 Gbps | UTP Cat 5e | 100m |
| **1000BASE-SX** | 1 Gbps | MMF | 550m |
| **1000BASE-LX** | 1 Gbps | SMF | 5 km |
| **10GBASE-T** | 10 Gbps | UTP Cat 6a | 100m |
| **10GBASE-SR** | 10 Gbps | MMF | 300m |
| **10GBASE-LR** | 10 Gbps | SMF | 10 km |
| **25GBASE-T** | 25 Gbps | Cat 8 | 30m |
| **40GBASE-SR4** | 40 Gbps | MMF (4-strand) | 100m |
| **100GBASE-SR4** | 100 Gbps | MMF | 100m |
| **100GBASE-LR4** | 100 Gbps | SMF | 10 km |
| **400GBASE-DR4** | 400 Gbps | SMF (4-strand) | 500m |
| **800GBASE** | 800 Gbps | SMF / DAC | — |

---

## 9. 프레임 캡처 / 분석 도구

### 9.1 tcpdump
```bash
sudo tcpdump -i en0 -e -nn -X
# -e: L2 헤더 표시 (MAC)
# -X: 16진수 + ASCII
```

### 9.2 Wireshark
- GUI, 프레임 구조 시각화
- 각 필드 클릭으로 raw byte 확인
- Display filter: `eth.type == 0x0800`

### 9.3 ethtool (Linux)
```bash
sudo ethtool -i eth0          # 드라이버 / 펌웨어
sudo ethtool eth0             # 속도 / duplex / link
sudo ethtool -S eth0          # 통계 (rx/tx errors, CRC)
sudo ethtool -k eth0          # offload 기능
```

---

## 10. 함정

### 함정 1 — Padding 무시
46 byte 미만 페이로드는 0 으로 padding. Sniff 시 인식.

### 함정 2 — Min frame (64) 의 의미
14 (header) + 46 (data) + 4 (FCS) = 64. 충돌 도메인의 RTT 보장 (옛 CSMA/CD).

### 함정 3 — 1500 MTU 가 최대 가정
Jumbo (9000), VPN (1400+), PPPoE (1492) 등 다양.

### 함정 4 — FCS 검증 위치
NIC HW 가 자동 검증 — tcpdump 가 보는 건 보통 FCS 떨어진 후. 깨진 프레임은 보고 어려움.

### 함정 5 — VLAN tag 의 위치
EtherType 앞이 아닌 SRC MAC 뒤. 802.1ad (QinQ) 는 2 개 tag.

### 함정 6 — Ethernet II 와 802.3 혼동
EtherType vs Length — 1536 (0x0600) 경계.

---

## 11. 학습 자료

- IEEE 802.3 표준 (무료) — https://standards.ieee.org/standard/802_3-2018.html
- **Ethernet: The Definitive Guide** (Charles Spurgeon)
- **TCP/IP Illustrated Vol.1** Ch. 3
- Wireshark sample captures — https://wiki.wireshark.org/SampleCaptures

---

## 12. 관련

- [[mac-address]] — 주소 필드 깊이
- [[error-detection-crc]] — FCS / CRC-32
- [[vlan-stp-bonding]] — 802.1Q tag 깊이
- [[csma-cd-ca]] — 매체 접근
- [[layer-2-data-link]] — 상위
