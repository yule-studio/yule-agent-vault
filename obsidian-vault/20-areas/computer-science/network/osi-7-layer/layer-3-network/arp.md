---
title: "ARP (Address Resolution Protocol)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T16:30:00+09:00
tags:
  - network
  - layer-3
  - arp
  - ndp
---

# ARP (Address Resolution Protocol)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | ARP / Gratuitous / RARP / Proxy / NDP / Spoofing |

**[[layer-3-network|↑ L3 Network]]** · **[[../osi-7-layer|↑↑ OSI 7]]**

> ARP 는 L2 와 L3 의 다리 — **IP 주소 → MAC 주소 해결**.
> 엄밀히 L2.5 이지만 IP 의 필수라 L3 와 함께 다룬다.

---

## 1. 한 줄 정의

같은 LAN 의 호스트에게 **"IP X 의 MAC 누구?"** 를 묻는 프로토콜. RFC 826 (1982).

---

## 2. 왜 ARP 가 필요한가

```
Host A 가 Host B (같은 LAN) 에게 데이터 보내려면:
- L3: IP 헤더의 Dest IP = B 의 IP (알고 있음)
- L2: Frame 의 Dest MAC = ?  ← 모름

→ ARP 로 IP → MAC 변환
```

호스트는 IP 만 알고 있지 LAN 안 누가 그 IP 인지 모름.

---

## 3. ARP 동작 흐름

```
1. Host A 가 패킷 보내려 함
2. ARP Cache 검색
   ├ 있음 → 사용
   └ 없음 → ARP Request 발송
3. ARP Request (Broadcast)
   "Who has 192.168.1.10? Tell 192.168.1.5"
   Dest MAC: FF:FF:FF:FF:FF:FF
4. 192.168.1.10 인 Host B 만 응답
   ARP Reply (Unicast to A)
   "192.168.1.10 is at AA:BB:CC:DD:EE:FF"
5. A 가 ARP Cache 에 저장 (보통 10-20 분)
6. 패킷 송신
```

---

## 4. ARP 패킷 구조

```
┌────────────────────────┬─────────────────┐
│ Hardware Type (2)      │ Protocol Type(2)│   HW: 1 = Ethernet
├──────────┬─────────────┼─────────────────┤   Proto: 0x0800 = IPv4
│ HW len(1)│ Proto len(1)│ Operation (2)   │   Op: 1 = Request, 2 = Reply
├──────────┴─────────────┴─────────────────┤
│ Sender HW (MAC) — 6 byte                 │
├──────────────────────────────────────────┤
│ Sender Protocol (IP) — 4 byte            │
├──────────────────────────────────────────┤
│ Target HW (MAC) — 6 byte                 │   Request 시 0
├──────────────────────────────────────────┤
│ Target Protocol (IP) — 4 byte            │
└──────────────────────────────────────────┘

EtherType = 0x0806
```

### 4.1 ARP Operation

| Op | 의미 |
| --- | --- |
| 1 | ARP Request |
| 2 | ARP Reply |
| 3 | RARP Request |
| 4 | RARP Reply |
| 25 | InARP |

---

## 5. ARP Cache

### macOS / Linux
```bash
arp -a                  # 전체
arp -n                  # 숫자만
ip neigh                # Linux 모던
ip neigh show 192.168.1.10
```

### Windows
```cmd
arp -a
```

### 상태 (Linux `ip neigh`)
- **REACHABLE** — 최근 확인됨
- **STALE** — 오래됨, 사용 시 재확인
- **DELAY** — 응답 기다림
- **PROBE** — 확인 중
- **FAILED** — 응답 없음
- **PERMANENT** — 정적 (수동 설정)

### Aging
- 보통 30 초 ~ 20 분
- 사용 시 갱신

---

## 6. Gratuitous ARP

자기 자신에 대한 ARP 를 broadcast — **자발적 ARP**.

```
"Who has 192.168.1.5? Tell 192.168.1.5"
(즉, 자기 IP 를 자기에게 묻기)
```

### 용도
- **IP 충돌 감지** — 부팅 시 자기 IP 보내봄, 응답 오면 충돌
- **MAC 변경 알림** — HA failover 시 새 노드가 자기 MAC 으로 brodcast
- **ARP Cache 강제 갱신** — 모든 호스트가 새 MAC 학습
- **VRRP / HSRP** failover

### 패킷
- Sender IP = Target IP (= 자기 IP)
- Sender MAC = 자기 MAC

---

## 7. Proxy ARP

다른 서브넷의 IP 에 대해 라우터가 자기 MAC 으로 응답.

```
PC (Sub A) → "Who has 10.0.0.5?" (Sub B 의 IP)
Router → "10.0.0.5 is at <Router MAC>"
PC → 패킷을 Router 로 보냄
Router → 실제 Sub B 의 10.0.0.5 로 forwarding
```

### 용도
- 잘못된 서브넷 마스크 보완 (옛 호스트)
- 모바일 IP

### 단점
- 호스트 의식 X → 라우팅 문제 진단 어려움
- 모던 환경에선 권장 X

---

## 8. RARP (Reverse ARP, RFC 903)

MAC → IP 해결. **자기 IP 를 모르는 디스크리스 클라이언트** 에서 사용.

```
"My MAC is X — what's my IP?"
RARP Server → "Your IP is Y"
```

- 1984 표준
- 현재 사장 — **DHCP** (1993) 가 대체

---

## 9. InARP (Inverse ARP)

- Frame Relay / ATM 에서
- DLCI / VPI/VCI ↔ IP 매핑

거의 사장.

---

## 10. ARP Spoofing / Cache Poisoning

### 10.1 공격

```
정상:
Victim → "Who has Gateway (192.168.1.1)?"
Gateway → "Gateway is at AA:BB:..."

공격:
Attacker → 가짜 ARP Reply 폭격
         "192.168.1.1 is at <Attacker MAC>"
Victim → ARP Cache 갱신 → Attacker 를 Gateway 로
→ 모든 트래픽이 Attacker 경유 (MITM)
```

### 10.2 도구
- arpspoof (dsniff)
- ettercap
- bettercap

### 10.3 방어
- **Static ARP entry** — 중요 호스트 (Gateway) 만 정적
- **Dynamic ARP Inspection (DAI)** — Switch 가 DHCP snooping binding 과 비교
- **ARPwatch** — ARP 변화 감지
- **VPN / mTLS** — L7 암호화
- **802.1X** — 포트 인증

---

## 11. NDP (Neighbor Discovery Protocol) — IPv6 의 ARP

IPv6 는 ARP 없이 **NDP (RFC 4861)** 사용. ICMPv6 메시지로:

| 메시지 | 용도 |
| --- | --- |
| **NS** (Neighbor Solicitation) | ARP Request |
| **NA** (Neighbor Advertisement) | ARP Reply |
| **RS** (Router Solicitation) | "라우터 누구?" |
| **RA** (Router Advertisement) | 라우터 광고 + Prefix |
| **Redirect** | "더 좋은 경로 있어" |

### NS 동작
- Solicited-Node Multicast 주소 사용
- broadcast 가 아닌 multicast → 효율 ↑

### SLAAC (Stateless Address Autoconfig)
- RA + Prefix → 호스트가 스스로 IPv6 주소 생성
- DHCP 없이 자동 설정

### NDP 보안
- **SEND** (Secure NDP) — 서명 추가, 사실상 사장
- **RA Guard**, **DHCPv6 Guard** — Switch 기능

---

## 12. 디버깅 — ARP 도구

### tcpdump
```bash
sudo tcpdump -i en0 -nn arp
# 출력:
# ARP, Request who-has 192.168.1.1 tell 192.168.1.5
# ARP, Reply 192.168.1.1 is-at aa:bb:cc:dd:ee:ff
```

### Wireshark
```
display filter: arp
arp.opcode == 1   (Request)
arp.opcode == 2   (Reply)
arp.duplicate-address-detected  (충돌)
```

### ping (간접)
```bash
ping 192.168.1.10
arp -a 192.168.1.10   # ping 후 캐시 확인
```

### arping (직접 ARP)
```bash
sudo arping -I en0 192.168.1.1
```

---

## 13. Cisco 명령

```
show arp                          # ARP 테이블
show ip arp                       # IP ARP
clear arp                         # 캐시 삭제
arp 192.168.1.1 0011.2233.4455 arpa   # 정적
```

---

## 14. 함정

### 함정 1 — ARP 가 작동 가정
방화벽이 ARP 차단 (드물지만) → 통신 X.

### 함정 2 — Cache stale
오래된 ARP 가 캐시에 → 새 MAC 못 찾음. Gratuitous ARP 로 갱신.

### 함정 3 — 같은 IP 두 호스트
ARP 응답이 충돌 — 캐시 flapping. `arping` 으로 진단.

### 함정 4 — IPv6 환경에서 ARP 사용 시도
IPv6 는 NDP. `ip -6 neigh`.

### 함정 5 — ARP 의 보안
인증 없음. Spoofing 쉬움. DAI 또는 mTLS.

### 함정 6 — Gratuitous ARP 의 broadcast 폭증
대형 LAN 에서 부담. 적당한 빈도.

---

## 15. 학습 자료

- RFC 826 (ARP), RFC 903 (RARP), RFC 4861 (NDP)
- **TCP/IP Illustrated Vol.1** Ch. 4
- Cisco "Dynamic ARP Inspection"
- HackTricks — ARP Spoofing

---

## 16. 관련

- [[icmp]] — ICMPv6 (NDP 의 기반)
- [[../layer-2-data-link/mac-address]] — MAC 주소
- [[../../ip/ipv4]], [[../../ip/ipv6]]
- [[layer-3-network]] — 상위
