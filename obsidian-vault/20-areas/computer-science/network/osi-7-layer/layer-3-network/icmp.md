---
title: "ICMP & ICMPv6 — ping, traceroute"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T16:35:00+09:00
tags:
  - network
  - layer-3
  - icmp
  - ping
  - traceroute
---

# ICMP & ICMPv6 — ping, traceroute

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | ICMP 타입 / ping / traceroute / Redirect / PMTUD |

**[[layer-3-network|↑ L3 Network]]** · **[[../osi-7-layer|↑↑ OSI 7]]**

---

## 1. 한 줄 정의

**I**nternet **C**ontrol **M**essage **P**rotocol. IP 의 상태 / 오류 / 진단 메시지 전달 프로토콜.
**ping**, **traceroute** 의 기반. RFC 792 (1981).

ICMP 는 L3 의 일부 — IP 헤더 위에 직접 (Protocol = 1).

---

## 2. ICMP 메시지 구조

```
[IP Header]
[ICMP Type (1)][Code (1)][Checksum (2)]
[Type-specific data]
```

### Type / Code 조합 (IPv4)

| Type | Code | 의미 |
| --- | --- | --- |
| **0** | 0 | Echo Reply (ping 응답) |
| **3** | 0 | Destination Network Unreachable |
| 3 | 1 | Destination Host Unreachable |
| 3 | 3 | Destination Port Unreachable |
| 3 | 4 | **Fragmentation Needed** (DF 설정 시) — PMTUD |
| 3 | 13 | Communication Administratively Prohibited |
| **5** | 0 | Redirect (네트워크) |
| 5 | 1 | Redirect (호스트) |
| **8** | 0 | Echo Request (ping) |
| **11** | 0 | **TTL Exceeded** (traceroute 의 핵심) |
| 11 | 1 | Fragment Reassembly Time Exceeded |
| **12** | 0 | Parameter Problem |
| 13 / 14 | — | Timestamp Request / Reply (옛) |
| 17 / 18 | — | Address Mask Request / Reply (옛) |

---

## 3. ping — Echo Request / Reply

### 3.1 동작

```
Host A → Echo Request (Type 8, ICMP id, seq, timestamp)
Host B → Echo Reply (Type 0, 같은 id/seq, 같은 payload)
Host A → RTT 측정
```

### 3.2 명령

```bash
ping google.com
ping -c 5 google.com         # 5 회만
ping -i 0.2 google.com       # 200 ms 간격
ping -s 1472 google.com      # 페이로드 크기 (Don't Fragment 와 같이 PMTUD)
ping -M do -s 1472 google.com    # Linux: DF 설정
ping6 google.com             # IPv6

# Windows
ping google.com -n 5 -l 1472
```

### 3.3 응답 정보

```
PING google.com (172.217.16.142): 56 data bytes
64 bytes from 172.217.16.142: icmp_seq=0 ttl=115 time=12.345 ms
```

- `bytes` — 페이로드 + ICMP 헤더 (8) = 56 + 8 = 64
- `ttl` — 응답의 TTL (라우터 hop 카운트 추정)
- `time` — RTT

### 3.4 ping 의 사용

- **연결성 확인**
- **RTT 측정**
- **MTU 확인** (DF + payload 크기)
- **패킷 손실 측정**
- **Latency 변동** (jitter)

### 3.5 ping 의 한계

- ICMP 차단된 호스트 ping 응답 X (TCP/UDP 는 가능)
- 진짜 부하 없음 — 응답 속도 ≠ 실제 서비스 속도
- DDoS 도구로 (ping flood / Smurf) — 일부 ICMP 정책으로 차단

---

## 4. traceroute / tracert

### 4.1 원리

```
TTL=1 패킷 → 첫 라우터가 TTL 0 → "TTL Exceeded" (ICMP Type 11) 응답
TTL=2 → 두 번째 라우터 응답
TTL=3 → ...
TTL=N → 목적지 도달 (또는 "Destination Unreachable")
```

각 hop 마다 라우터 IP 알아냄.

### 4.2 명령

```bash
# macOS / Linux — UDP 기반 (높은 포트)
traceroute google.com
traceroute -I google.com     # ICMP Echo 사용

# Linux mtr — ping + traceroute 통합
mtr google.com

# Windows — ICMP Echo 기본
tracert google.com
```

### 4.3 패킷 종류

| OS | 기본 방식 |
| --- | --- |
| macOS / Linux traceroute | UDP (33434~ 포트) |
| Windows tracert | ICMP Echo |
| `traceroute -T` | TCP SYN (방화벽 통과) |

### 4.4 출력 예

```
 1  192.168.1.1   1.2 ms
 2  10.0.0.1      5.1 ms
 3  172.16.0.1   10.3 ms
 *  *  *          (타임아웃)
 5  8.8.8.8      15.2 ms
```

- `*` — 라우터가 응답 안 함 (rate limit / 정책)

### 4.5 함정

- **비대칭 경로** — 보내는 / 받는 경로 다를 수 있음 → traceroute 는 송신 경로만
- **MPLS** — MPLS 내부 라우터는 안 보일 수 있음
- **Rate limiting** — 라우터가 ICMP 응답 제한
- **방화벽** — UDP traceroute 차단 → ICMP 또는 TCP 시도

---

## 5. Destination Unreachable (Type 3)

라우터 / 호스트가 패킷 전달 불가 시:

| Code | 의미 |
| --- | --- |
| 0 | Network Unreachable |
| 1 | Host Unreachable |
| 2 | Protocol Unreachable |
| 3 | **Port Unreachable** (UDP 포트 닫힘) |
| 4 | **Fragmentation Needed** (DF, PMTUD 의 핵심) |
| 6 | Destination Network Unknown |
| 7 | Destination Host Unknown |
| 9 / 10 | Network/Host Administratively Prohibited |
| 13 | Communication Administratively Prohibited |

### 5.1 UDP Port Unreachable

UDP 는 비연결 — 닫힌 포트로 보내면 ICMP Type 3 Code 3 응답.

→ UDP 의 traceroute 가 목적지 도달 신호로 사용.

### 5.2 Fragmentation Needed (DF set)

PMTUD 의 핵심.

```
Sender → Packet (size 1500, DF=1)
Router → "1400 까지만 OK" → ICMP Type 3 Code 4
Sender → 1400 으로 재시도 (또는 더 작게)
```

방화벽이 ICMP 차단 시 → **PMTUD 실패** → "ICMP black hole".

---

## 6. Redirect (Type 5)

라우터가 호스트에게 "더 좋은 경로 있어":

```
Host → Router A 로 보냄 (default GW)
Router A: "이건 Router B 가 더 가까움"
Router A → Host: ICMP Redirect "다음부턴 Router B 에게"
Host → 라우팅 테이블 갱신
```

### 보안 위험
- 악의적 Redirect 로 트래픽 가로채기
- 모던 호스트는 보통 비활성 (`net.ipv4.conf.all.accept_redirects=0`)

---

## 7. ICMP Source Quench (deprecated)

옛 혼잡 알림 — 송신 측에게 "천천히 보내".

- 라우터가 보내야 하지만 비효율
- RFC 6633 으로 deprecated
- ECN (Explicit Congestion Notification) 으로 대체

---

## 8. ICMPv6 (RFC 4443)

IPv6 의 ICMP. IP 헤더 위에 Next Header = 58.

### 8.1 ICMPv6 메시지 종류

| Type | 의미 |
| --- | --- |
| 1 | Destination Unreachable |
| 2 | Packet Too Big (PMTUD — IPv6 는 라우터가 단편화 X) |
| 3 | Time Exceeded |
| 4 | Parameter Problem |
| 128 | Echo Request |
| 129 | Echo Reply |
| 133 | **Router Solicitation** (NDP) |
| 134 | **Router Advertisement** |
| 135 | **Neighbor Solicitation** (ARP 대체) |
| 136 | **Neighbor Advertisement** |
| 137 | Redirect |

### 8.2 차이점

- **NDP** 가 ICMPv6 위 — ARP 대체
- **MLD** (Multicast Listener Discovery) — IGMP 대체
- **PMTUD 필수** — IPv6 라우터는 단편화 X, 호스트가 처리

---

## 9. ICMP Tunneling

ICMP 페이로드에 임의 데이터 → 데이터 유출 / 우회.

- 도구: ptunnel, icmpsh
- 방화벽이 ping 만 허용 시 우회
- 방어: 페이로드 검사, ICMP 정책

---

## 10. ICMP Flood (DDoS)

### 10.1 Ping Flood
다수 호스트에서 ICMP Echo 폭격.

### 10.2 Smurf Attack (옛)
- Broadcast 주소로 spoofed ping
- 모든 호스트가 victim 에 응답
- RFC 2644 — broadcast ping 비활성

### 10.3 Ping of Death (옛)
- 큰 ICMP (>65535 byte) 으로 OS 크래시
- 1996 — 모든 OS 패치됨

---

## 11. 방화벽과 ICMP

### 11.1 차단 시 문제
- ping / traceroute 안 됨
- **PMTUD 실패** — TCP 가 큰 패킷 보내고 다음부터 안 옴
- 진단 어려움

### 11.2 권장 정책
- **Echo Request** — 차단 OK (보안)
- **Type 3 / Type 11** — 허용 (PMTUD / traceroute)
- **NDP (IPv6)** — 반드시 허용

```
# iptables (Linux)
iptables -A INPUT -p icmp --icmp-type fragmentation-needed -j ACCEPT
iptables -A INPUT -p icmp --icmp-type echo-request -j DROP
iptables -A INPUT -p icmp -j DROP    # 나머지 차단

# IPv6
ip6tables -A INPUT -p icmpv6 -j ACCEPT   # 거의 모두 허용 필요
```

---

## 12. ICMP 와 TTL

각 라우터 통과 시 TTL -= 1. 0 되면 Type 11 응답 후 패킷 폐기.

### TTL 의 기본값
- Linux: 64
- Windows: 128
- Cisco IOS: 255
- macOS: 64

→ 응답 TTL 보고 OS 추정 (fingerprinting).

---

## 13. 함정

### 함정 1 — ICMP 차단으로 PMTUD 실패
TCP 가 small MSS 로 협상해도 후속 패킷 손실 → "ICMP black hole".

### 함정 2 — ping = 연결성 가정
TCP/UDP 서비스는 OK 인데 ICMP 차단된 경우 있음.

### 함정 3 — traceroute 의 비대칭 경로
양방향 다를 수 있음. mtr 도 같은 문제.

### 함정 4 — ICMP Redirect 위험
공격 도구. 비활성.

### 함정 5 — IPv6 의 ICMPv6 차단
NDP 안 동작 → 통신 자체 X.

### 함정 6 — Source Quench 활성
deprecated. 무시.

---

## 14. 학습 자료

- RFC 792 (ICMP), RFC 4443 (ICMPv6), RFC 1191 (PMTUD)
- **TCP/IP Illustrated Vol.1** Ch. 6-8
- "Smurf Attack" (CERT)
- Cisco "ICMP Best Practices"

---

## 15. 관련

- [[arp]] — IPv4 의 IP-MAC, IPv6 는 ICMPv6 NDP
- [[fragmentation-mtu]] — PMTUD 와 ICMP Type 3/4
- [[../../tools/tools|↗ ping / traceroute 도구]]
- [[layer-3-network]] — 상위
