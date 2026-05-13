---
title: "ping / traceroute / mtr — 경로 / 지연"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T09:25:00+09:00
tags:
  - network
  - tools
  - ping
  - traceroute
---

# ping / traceroute / mtr — 경로 / 지연

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | ICMP / TTL / hop |

**[[tools|↑ 도구 hub]]** · **[[../network|↑↑ network hub]]**

---

## 1. ping — ICMP Echo

### 기본
```bash
ping example.com
ping6 example.com           # IPv6

# Linux 기본 — 무한
# macOS 기본 — 무한, Ctrl+C 종료
```

### 옵션
```bash
ping -c 4 example.com         # 4 번
ping -i 0.5 example.com       # 0.5 초 간격
ping -s 1400 example.com      # payload 1400 byte
ping -W 2 example.com         # timeout 2 초
ping -f example.com           # flood (root)
ping -t 64 example.com        # TTL 64
ping -I eth0 example.com      # interface 지정
```

### 결과 해석
```
PING example.com (1.2.3.4): 56 data bytes
64 bytes from 1.2.3.4: icmp_seq=0 ttl=64 time=10.234 ms

--- example.com ping statistics ---
4 packets transmitted, 4 received, 0% packet loss
round-trip min/avg/max/stddev = 10.123/10.500/11.234/0.456 ms
```

- **time** — RTT
- **ttl** — 응답 packet 의 TTL (어디서 왔는지)
- **packet loss** — 손실률
- **stddev** — 변동 (지터)

### 한계
- ICMP 차단 — 일부 server / 방화벽 ping X
- ping 만 OK ≠ 서비스 OK (port 막힘 가능)
- ping fail ≠ server down (ICMP 차단)

### 대안 — hping3
```bash
sudo hping3 -S -p 443 example.com    # TCP SYN ping
```

→ ICMP 차단된 서버 ping.

---

## 2. traceroute / tracert

### 원리
- TTL 1 → 첫 hop 응답 (ICMP Time Exceeded)
- TTL 2 → 두 번째 hop
- ...
- 목적지 도달 (port unreachable / TCP ACK)

### Linux / macOS
```bash
traceroute example.com
traceroute -n example.com         # DNS X
traceroute -m 30 example.com      # max hops 30
traceroute -q 1 example.com       # 1 probe per hop
traceroute -T -p 443 example.com  # TCP traceroute (port 443)
traceroute -I example.com         # ICMP echo (Windows tracert 와 같음)
```

### Windows
```bash
tracert example.com               # ICMP 기본
```

### 결과
```
traceroute to example.com (1.2.3.4), 64 hops max
 1  192.168.1.1   1.234 ms   1.123 ms   1.456 ms
 2  10.0.0.1      5.234 ms   5.345 ms   5.123 ms
 3  isp-router    10.234 ms  10.345 ms  10.456 ms
 4  *  *  *                                  ← 막힘
 5  cdn-edge      15.234 ms  15.345 ms  15.123 ms
 6  example.com   16.234 ms  16.345 ms  16.456 ms
```

### * * * 의 의미
- ICMP Time Exceeded 응답 없음
- 방화벽 차단
- Hop 자체는 있을 수 있음

### Probe 방법
- **UDP** (Unix 기본) — 큰 port (33434+)
- **ICMP** (Windows / `-I`) — Echo
- **TCP** (`-T`) — port (443) — 방화벽 우회

---

## 3. mtr — ping + traceroute 결합

### 정의
- 1997, Roger Wolff
- 각 hop 마다 통계
- 실시간 / continuous

### 기본
```bash
mtr example.com
```

### 결과 (interactive)
```
                            Packets               Pings
Host                       Loss%  Snt   Last   Avg  Best  Wrst StDev
1. 192.168.1.1              0.0%   10    1.2    1.5   1.1   2.3   0.4
2. 10.0.0.1                 0.0%   10    5.3    5.5   5.2   6.0   0.3
3. isp-router              10.0%   10   10.2   12.0  10.0  20.0   3.2    ← 손실!
4. cdn-edge                 0.0%   10   15.4   15.5  15.2  16.0   0.3
5. example.com              0.0%   10   16.2   16.5  16.1  17.0   0.3
```

### 옵션
```bash
mtr --report --report-cycles 100 example.com    # 100 회 + 종료
mtr -T -P 443 example.com                        # TCP
mtr -n example.com                               # no DNS
mtr --json example.com
```

### 사용
- Latency 변동 / 손실 분석
- 어느 hop 이 병목?
- ISP / CDN / origin 중 어디?

---

## 4. pathping (Windows)

### 정의
- ping + traceroute (Windows native)

```bash
pathping example.com
```

### 결과
- 각 hop 의 latency / loss
- 일정 시간 측정 후 통계

---

## 5. ICMP 의 응답 코드

| Type | Code | 의미 |
| --- | --- | --- |
| 0 | 0 | Echo Reply (ping 응답) |
| 3 | 0 | Destination Network Unreachable |
| 3 | 1 | Destination Host Unreachable |
| 3 | 3 | Port Unreachable |
| 3 | 4 | Fragmentation Needed |
| 8 | 0 | Echo Request (ping) |
| 11 | 0 | Time Exceeded (TTL=0) |

### Fragmentation Needed (PMTU)
- 큰 패킷 + DF flag → ICMP 응답
- Path MTU Discovery

자세히 → [[../ip/mtu]] (TBD)

---

## 6. 흔한 시나리오

### 6.1 연결 확인
```bash
ping -c 4 example.com         # 응답 있나
ping6 example.com             # IPv6 도
```

### 6.2 ISP / CDN / Origin 어디 문제?
```bash
mtr example.com
# Hop 별 loss / latency 보고
# - Hop 1-2 ISP
# - Hop 3-5 backbone
# - Hop 6+ CDN / origin
```

### 6.3 방화벽이 ICMP 차단?
```bash
# TCP traceroute
traceroute -T -p 443 example.com
mtr -T -P 443 example.com
```

### 6.4 RTT 변동 (지터)
```bash
ping -c 100 example.com | awk '/time=/ { print }' | sort
# 또는 mtr StDev
```

### 6.5 다른 region 의 latency
```bash
# AWS EC2 — region 별 ping
ssh us-east-1 ping example.com
ssh eu-west-1 ping example.com
ssh ap-northeast-1 ping example.com
```

---

## 7. PMTU 발견

### 원리
- 큰 패킷 + DF (Don't Fragment) → ICMP Fragmentation Needed
- 작게 줄여 재전송

### tracepath
```bash
tracepath example.com
# Hop 별 MTU
```

### ping 으로
```bash
ping -M do -s 1472 example.com    # 1472 + 28 header = 1500 (Ethernet MTU)
ping -M do -s 1473 example.com    # 1473 — fragment needed
```

→ Path MTU 발견 (IPv4: 1500, VPN: 1400, Tunnel: 더 작음).

---

## 8. 함정

### 함정 1 — ICMP 차단
일부 서버 — ping 응답 X. TCP traceroute (`-T`) 권장.

### 함정 2 — Async path
요청 / 응답 경로 다름. Traceroute — 요청 경로만 봄.
응답은 reverse traceroute 도구 필요.

### 함정 3 — Load balancer 의 hop
ECMP — 다른 hop 매번. Traceroute 결과 일관성 없음.

### 함정 4 — * 의 false alarm
ICMP rate limit / 방화벽 — 실제 hop 은 동작. 무시 OK.

### 함정 5 — mtr 의 RTT 정확
일부 라우터 — ICMP TTL Exceeded 의 우선순위 낮음 → 측정 latency 부정확.
But TCP latency 도 비슷한 경향.

### 함정 6 — IPv6 의 차이
일부 NS / route — IPv6 없음. ping6 / traceroute6.

---

## 9. 학습 자료

- "TCP/IP Illustrated"
- mtr docs
- "Internet Routing in Practice"

---

## 10. 관련

- [[tools]] — Hub
- [[ss-netstat-tcp-tools]] — local socket
- [[../ip/ipv4-vs-ipv6]] — ping / ping6
- [[../routing/bgp]] — backbone routing
