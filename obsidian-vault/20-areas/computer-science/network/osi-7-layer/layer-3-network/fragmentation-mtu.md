---
title: "단편화 (Fragmentation) & MTU / PMTUD"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T16:40:00+09:00
tags:
  - network
  - layer-3
  - mtu
  - fragmentation
  - pmtud
---

# 단편화 (Fragmentation) & MTU / PMTUD

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | IPv4 단편화 / PMTUD / IPv6 차이 / ICMP black hole |

**[[layer-3-network|↑ L3 Network]]** · **[[../osi-7-layer|↑↑ OSI 7]]**

---

## 1. 한 줄 정의

큰 IP 패킷이 MTU 더 작은 링크 통과 시 **여러 작은 단편 (Fragment)** 으로 쪼개기.
**수신 측에서 재조립**. IPv4 는 라우터가, IPv6 는 송신자가 처리.

---

## 2. MTU (Maximum Transmission Unit)

```
MTU = L2 가 한 번에 실어 나를 수 있는 L3 페이로드 최대 byte
```

### 매체별 MTU

| 매체 | MTU |
| --- | --- |
| **Ethernet** (표준) | **1500** |
| Ethernet **Jumbo** | 9000 |
| Wi-Fi (802.11) | 2304 (실제론 1500) |
| **PPPoE** | 1492 (8 byte 헤더) |
| GRE Tunnel | 1476 |
| IPsec Tunnel | 1400 ~ 1438 |
| MPLS | 1500 - N×4 (label 마다 4) |
| Token Ring | 4464 |
| FDDI | 4352 |
| Localhost (loopback) | 65536 |
| WireGuard | 1420 (1500 - 80) |

### MTU 확인

```bash
# Linux
ip link show eth0 | grep mtu

# macOS
ifconfig en0 | grep mtu

# Windows
netsh interface ipv4 show subinterfaces
```

---

## 3. IPv4 단편화 (Fragmentation)

### 3.1 IPv4 헤더 필드

```
[Identification (16)] — 같은 원본 패킷의 단편 묶음
[Flags (3)]
   bit 0: Reserved (0)
   bit 1: DF (Don't Fragment)
   bit 2: MF (More Fragments)
[Fragment Offset (13)] — 8-byte 단위 (최대 65528 byte)
```

### 3.2 단편화 예

```
원본 패킷: 4000 byte (IP 헤더 20 + 데이터 3980)
경로 MTU: 1500

→ 단편 1: offset=0, length=1480, MF=1
→ 단편 2: offset=185 (×8), length=1480, MF=1
→ 단편 3: offset=370, length=1020, MF=0

(모두 같은 Identification, IP 헤더 복제)
```

### 3.3 재조립

수신 호스트가:
1. 같은 Identification + Src/Dst IP 의 단편 모음
2. Offset 으로 순서 결정
3. MF=0 의 단편으로 끝 식별
4. 타임아웃 (60 초) 시 일부 missing 이면 폐기

---

## 4. 단편화의 비용 / 위험

### 4.1 라우터 부담
- 헤더 복제 + 검사
- 라우터 CPU 부하

### 4.2 재조립 비용
- 호스트 메모리 (단편 버퍼)
- DoS — "Teardrop attack" — 잘못된 offset 으로 OS 크래시 (옛 Windows)

### 4.3 손실 시 전체 폐기
- 한 단편 손실 → 모든 단편 폐기 → 재전송 모두
- 작은 패킷보다 효율 ↓

### 4.4 NAT / 방화벽
- 단편의 첫 번째만 L4 헤더 있음 → 후속 단편의 정책 결정 어려움
- 일부 NAT 가 단편 폐기

### 4.5 PMTUD 가 권장되는 이유
단편화 자체를 피하는 것이 가장 좋음.

---

## 5. PMTUD (Path MTU Discovery, RFC 1191)

### 5.1 동작

```
1. 송신자: 큰 패킷 + DF=1 (Don't Fragment)
2. 라우터: MTU 초과 + DF 설정
3. 라우터: 패킷 폐기 + ICMP Type 3 Code 4 "Fragmentation Needed, MTU=1400"
4. 송신자: ICMP 받음 → MTU 1400 으로 재시도
5. 더 작은 MTU 만나면 반복
6. 종단까지 도달 → Path MTU 결정
```

### 5.2 TCP 의 MSS

PMTUD 와 함께:
- TCP SYN 의 MSS option → 자기 MSS 알림
- 양쪽이 작은 쪽 선택
- 그 후 PMTUD 로 더 작아질 수 있음

```
MSS = MTU - 20 (IP) - 20 (TCP) = 1460  (Ethernet 기준)
```

### 5.3 PMTU 캐시

OS 가 (목적지, PMTU) 캐시 — 10 분 후 다시 시도 (큰 MTU 가능성).

---

## 6. ICMP Black Hole

PMTUD 실패의 원인:

```
방화벽이 ICMP Type 3 Code 4 차단
→ 송신자가 "Frag needed" 신호 못 받음
→ 큰 패킷 계속 폐기됨
→ 작은 패킷 (SYN, ACK) 만 통과
→ SYN 은 성공, 데이터는 stuck
```

### 증상
- 사이트 connect 는 됨 (TCP 3-way OK)
- 페이지 일부만 로드 / 멈춤
- 큰 payload 의 응답 받을 때 stuck

### 진단
```bash
# 큰 패킷 ping (DF)
ping -M do -s 1472 google.com    # Linux
ping -D -s 1472 google.com       # macOS

# 결과:
# "Frag needed and DF set" → MTU 알 수 있음
# 응답 없음 → ICMP 차단 (Black Hole)
```

### 해결
- 방화벽: ICMP Type 3 Code 4 허용
- TCP MSS Clamping (라우터 / NAT 가 MSS 강제 축소)

---

## 7. MSS Clamping

라우터 / VPN 게이트웨이가 SYN 의 MSS option 을 강제로 작게 설정:

```bash
# iptables
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --clamp-mss-to-pmtu

# 명시적
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --set-mss 1400
```

VPN / PPPoE / GRE 환경에서 흔히 사용.

---

## 8. IPv6 와 단편화

### 8.1 차이점

| 측면 | IPv4 | IPv6 |
| --- | --- | --- |
| 단편화 주체 | 라우터 + 송신자 | **송신자만** |
| 헤더 필드 | Identification / Flags / Offset | **확장 헤더 (Fragment Header)** |
| 최소 MTU | 576 (1280 권장) | **1280 (필수)** |
| 라우터 동작 | 단편화 / 폐기 + ICMP | 폐기 + ICMPv6 Type 2 |
| PMTUD | 권장 | **필수** |

### 8.2 IPv6 Fragment Header

```
[Next Header (1)][Reserved (1)][Fragment Offset (13)][Res 2][M flag (1)]
[Identification (32)]
```

기본 IPv6 헤더에는 단편화 필드 없음 — 필요 시 확장 헤더 추가.

### 8.3 IPv6 PMTUD

- 모든 라우터가 MTU 초과 시 **ICMPv6 Type 2 (Packet Too Big)** 응답
- 송신자는 이 정보로 MSS 조정
- ICMPv6 차단 시 **연결 자체 불가** — IPv4 보다 치명적

---

## 9. Jumbo Frame 의 함정

```
PC1 (MTU 9000) ─ Switch ─ Router (MTU 1500) ─ Internet
```

PC1 이 9000 byte 보냄 → Router 가 단편화 또는 DF 면 폐기 → 흐름 깨짐.

→ Jumbo 는 **완전한 end-to-end** 일 때만 의미. 모든 hop 이 동일 MTU.

---

## 10. NAT 와 MTU

NAT 가 추가 헤더 (IPsec, GRE, VXLAN) 캡슐화 시 effective MTU 감소:

| 캡슐화 | 오버헤드 |
| --- | --- |
| IPsec ESP (transport) | 20-40 byte |
| IPsec ESP (tunnel) | 40-60 byte |
| GRE | 24 byte |
| VXLAN | 50 byte (UDP + VXLAN) |
| L2TP | 12 byte |
| PPPoE | 8 byte |
| MPLS | 4 byte × labels |

→ 안쪽 패킷 MTU 를 그만큼 줄여야.

---

## 11. 실용 — MTU 결정

### 11.1 단계
1. `ip link show` 로 인터페이스 MTU
2. `ping -M do -s N` 으로 점진적으로 줄여가며 응답 보는 max N 찾기
3. ICMP type 3/4 응답 보이면 그 값이 PMTU

### 11.2 트래픽별 권장

| 환경 | MTU |
| --- | --- |
| 일반 Internet | 1500 |
| PPPoE (한국 ISP 일부) | 1492 |
| IPsec VPN | 1400 |
| WireGuard | 1420 |
| Docker (기본 bridge) | 1500 |
| Kubernetes (Flannel VXLAN) | 1450 |
| 데이터센터 (Jumbo) | 9000 |

---

## 12. tcpdump 로 단편화 확인

```bash
sudo tcpdump -i en0 -nn 'ip[6:2] & 0x3fff != 0'
# bit 13-15: Flags + Offset
# 단편화된 패킷만 표시
```

Wireshark display filter: `ip.flags.mf == 1 or ip.frag_offset > 0`.

---

## 13. 함정

### 함정 1 — DF=1 + ICMP 차단
PMTUD 실패. ICMP type 3/4 허용 필수.

### 함정 2 — VPN 후 MTU 조정 안 함
패킷 손실 / 느림. MSS Clamping.

### 함정 3 — 일부 hop 만 Jumbo
End-to-end 아니면 단편화 또는 폐기.

### 함정 4 — IPv6 의 ICMPv6 차단
PMTUD 의존 → 통신 실패.

### 함정 5 — TCP 만 MSS 조정
UDP / ICMP 는 PMTUD 만 의지. UDP DNS 큰 응답 (DNSSEC) 단편화 문제.

### 함정 6 — DNS UDP 의 단편화
EDNS0 buffer 4096 + DNSSEC → 단편화. 일부 방화벽이 차단. TCP fallback 또는 DoH/DoT.

---

## 14. 학습 자료

- RFC 791 (IPv4), RFC 8200 (IPv6), RFC 1191 (PMTUD), RFC 4821 (Packetization Layer PMTUD)
- **TCP/IP Illustrated Vol.1** Ch. 10
- Cloudflare "MTU and the curious case of Path MTU Discovery"

---

## 15. 관련

- [[icmp]] — Type 3 Code 4, ICMPv6 Type 2
- [[../layer-4-transport/tcp]] — MSS / MSS Clamping
- [[../layer-2-data-link/ethernet-frame]] — Jumbo Frame
- [[layer-3-network]] — 상위
