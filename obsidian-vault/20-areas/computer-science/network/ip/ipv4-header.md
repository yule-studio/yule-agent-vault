---
title: "IPv4 헤더 (Header Structure)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T19:00:00+09:00
tags:
  - network
  - ipv4
  - ip-header
---

# IPv4 헤더 (Header Structure)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 헤더 필드 / DSCP/ECN / Options / Protocol Numbers |

**[[ip|↑ IP]]** · **[[../network|↑↑ network hub]]**

---

## 1. 헤더 전체

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version|  IHL  |DSCP   |ECN |          Total Length             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Identification        |Flags|      Fragment Offset    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Time to Live |    Protocol   |         Header Checksum       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Source Address                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Destination Address                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options                    |    Padding    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Data                                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

기본 **20 byte** + Options (최대 40 byte) = **60 byte 최대**.

---

## 2. 필드별 상세

### 2.1 Version (4 bit)
- IPv4 = `4`, IPv6 = `6`
- 라우터가 먼저 보고 어느 처리할지 결정

### 2.2 IHL — Internet Header Length (4 bit)
- 헤더 길이 / 4 (32-bit word 수)
- 최소 5 (= 20 byte), 최대 15 (= 60 byte)
- Options 가 있으면 5 초과

### 2.3 DSCP — Differentiated Services (6 bit)

QoS 마킹:
| Class | 값 | 용도 |
| --- | --- | --- |
| **CS0** | 0 | Best Effort |
| **CS1** | 8 | Low priority data |
| **AF11-43** | 10-38 | Assured Forwarding (4 class × 3 drop) |
| **EF** | 46 | Expedited Forwarding (VoIP) |
| **CS6** | 48 | Network Control (라우팅 프로토콜) |
| **CS7** | 56 | Reserved |

이전엔 **ToS** (Type of Service) 였고 일부 비트는 호환성 유지.

### 2.4 ECN — Explicit Congestion Notification (2 bit)

| 값 | 의미 |
| --- | --- |
| 00 | Not-ECT (ECN 미지원) |
| 01 / 10 | ECT(0/1) — ECN 지원 |
| 11 | CE (Congestion Experienced) — 라우터가 표시 |

자세히 → [[../tcp/congestion-control#9 ECN]]

### 2.5 Total Length (16 bit)
- IP 헤더 + 데이터 전체 byte
- 최대 **65535 byte** (실제 MTU 가 더 작음)

### 2.6 Identification (16 bit)
- 단편화된 패킷들의 묶음 ID
- 같은 (Src IP, Dst IP, Protocol, ID) = 같은 원본 패킷

### 2.7 Flags (3 bit)

| bit | 이름 | 의미 |
| --- | --- | --- |
| 0 | Reserved | 항상 0 |
| 1 | **DF** (Don't Fragment) | 1 = 단편화 금지 → MTU 초과 시 폐기 + ICMP |
| 2 | **MF** (More Fragments) | 1 = 더 있음, 0 = 마지막 |

### 2.8 Fragment Offset (13 bit)
- 원본 패킷에서 이 단편의 시작 위치 / 8
- 최대 65528 byte (8192 × 8)

### 2.9 TTL — Time to Live (8 bit)
- 각 hop 마다 -1 → 0 되면 폐기 + ICMP TTL Exceeded
- 라우팅 루프 방지
- OS 기본:
  - Linux: 64
  - Windows: 128
  - Cisco IOS: 255
- 응답 TTL 로 OS 추정 가능 (fingerprinting)

### 2.10 Protocol (8 bit)

상위 L4 프로토콜:

| 값 | 프로토콜 |
| --- | --- |
| **1** | ICMP |
| 2 | IGMP |
| **6** | TCP |
| **17** | UDP |
| 41 | IPv6 over IPv4 (Teredo, 6in4) |
| 47 | GRE |
| 50 | ESP (IPsec) |
| 51 | AH (IPsec) |
| 58 | ICMPv6 |
| 89 | OSPF |
| 132 | SCTP |

전체 → IANA Protocol Numbers.

### 2.11 Header Checksum (16 bit)

- 헤더만 (데이터 X) 의 1's complement
- **매 hop 마다 재계산** (TTL 감소로 헤더 변경)
- IPv6 에선 제거 — 비용 절감

### 2.12 Source Address (32 bit)
출발 IP. NAT 가 변경.

### 2.13 Destination Address (32 bit)
목적 IP.

### 2.14 Options (가변, 0-40 byte)

거의 사용 X — 일부 보안 / 진단:

| Option | 용도 |
| --- | --- |
| **Record Route** | 경로 기록 (8 hop 한계) |
| **Strict Source Routing** | 명시 경로 (보안 위험 — 차단) |
| **Loose Source Routing** | 일부 경유 |
| **Timestamp** | hop 별 시간 |
| **Router Alert** | 라우터가 검사 (IGMP, MPLS) |

> 대부분 방화벽이 차단. Source Routing 은 spoofing 위험 → IPv6 도 SRH 만 일부.

### 2.15 Padding
Options 가 4 byte 정렬 아니면 0 으로 채움.

---

## 3. Pseudo Header (L4 checksum 용)

TCP / UDP 의 checksum 은 IP 의 일부 필드 포함:

```
+--------+--------+--------+--------+
|           Source IP               |
+--------+--------+--------+--------+
|         Destination IP            |
+--------+--------+--------+--------+
|  zero  | Proto  |   TCP/UDP Length|
+--------+--------+--------+--------+
```

NAT 가 IP / Port 변경 시 L4 checksum 재계산 필수.

---

## 4. tcpdump 로 헤더 보기

```bash
sudo tcpdump -i en0 -nn -v
# 출력:
# 12:00:01.123 IP (tos 0x0, ttl 64, id 12345, offset 0, flags [DF], proto TCP (6), length 60)
#   192.168.1.5.49152 > 1.2.3.4.443: Flags [S], seq 1000, win 65535, ...
```

Wireshark display filter:
```
ip.version == 4
ip.flags.df == 1
ip.proto == 6
ip.ttl < 30
ip.dsfield.dscp == 46     (EF/VoIP)
```

---

## 5. 함정

### 함정 1 — IPv4 ID 의 wrap
빠른 송신 시 16 bit ID 가 짧은 시간에 wrap → 단편 혼동. 라우터가 단편화하면 위험.

### 함정 2 — DF + ICMP 차단
PMTUD 실패 → "ICMP black hole". 자세히 → [[../osi-7-layer/layer-3-network/fragmentation-mtu]].

### 함정 3 — Options 의 보안 위험
Source Routing 으로 spoofing → 대부분 라우터 차단.

### 함정 4 — Header Checksum 만 검증
데이터는 검증 X — TCP/UDP checksum 또는 L2 CRC 가 보호.

### 함정 5 — Protocol 47 (GRE), 50 (ESP) NAT 통과 어려움
포트 개념 없음 → NAT-T (UDP 캡슐화) 필요.

### 함정 6 — DSCP 의 신뢰
ISP / 인터넷이 DSCP 무시. 사내 망에서만.

---

## 6. 학습 자료

- RFC 791 (IPv4), RFC 2474 (DSCP), RFC 3168 (ECN)
- **TCP/IP Illustrated Vol.1** Ch. 5
- IANA Protocol Numbers — https://www.iana.org/assignments/protocol-numbers/
- Wireshark "IPv4" dissector 문서

---

## 7. 관련

- [[ip]] — IP hub
- [[ipv6-header]] — IPv6 헤더 비교
- [[../osi-7-layer/layer-3-network/fragmentation-mtu]] — Flags / Fragment Offset 활용
- [[../osi-7-layer/layer-3-network/icmp]] — Protocol = 1
