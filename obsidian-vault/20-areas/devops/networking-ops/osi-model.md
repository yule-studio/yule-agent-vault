---
title: "OSI / TCP-IP model — L4 vs L7"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:42:00+09:00
tags: [devops, networking-ops, osi]
---

# OSI / TCP-IP model — L4 vs L7

**[[networking-ops|↑ networking-ops]]**

---

## 1. 7 layer

| Layer | 이름 | 책임 | 도구 / 프로토콜 | DevOps 에서 |
| --- | --- | --- | --- | --- |
| L7 | Application | app 데이터 | HTTP, DNS, gRPC | nginx, ALB |
| L6 | Presentation | 변환 / 암호화 | TLS, JSON | (L7 안에 합침) |
| L5 | Session | 세션 | (TLS handshake) | (L7) |
| L4 | Transport | 종단간 | TCP, UDP | NLB, HAProxy |
| L3 | Network | 라우팅 | IP, ICMP | router, VPC route |
| L2 | Data link | 인접 노드 | Ethernet, MAC | switch |
| L1 | Physical | 신호 | 케이블, RF | hardware |

→ TCP/IP 모델은 4 layer (Application / Transport / Internet / Link).

---

## 2. L4 vs L7 LB (★ 면접 단골)

### L4

```
[client] ─TCP→ [L4 LB] ─TCP→ [backend]
                    ↑ IP + port 만 봄
                    payload (HTTP) 안 봄
```

- 빠름 (kernel 단)
- HTTP 헤더 / path 못 봄 → routing 불가
- 예: NLB, HAProxy mode TCP, IPVS, LVS, F5 BIG-IP

### L7

```
[client] ─HTTP→ [L7 LB] ─HTTP→ [backend]
                     ↑ URL / header / cookie 분석
                       → /api/users → backend A
                       → /static    → backend B
```

- 느림 (proxy 처리)
- HTTP routing / TLS termination / header 조작
- 예: ALB, nginx, Envoy, Traefik, HAProxy mode HTTP

---

## 3. 언제 어떤 거

| 상황 | LB |
| --- | --- |
| TCP/UDP (PostgreSQL, Redis, Kafka) | L4 (NLB) |
| 단순 HTTP, 고성능 | L4 |
| Path-based routing (`/api`, `/web`) | L7 (ALB / nginx) |
| TLS termination | L7 |
| WebSocket / gRPC | L7 (L4 도 가능) |
| static IP 필요 | NLB |
| WAF 통합 필요 | L7 (ALB + AWS WAF) |
| 글로벌 | GLB / CloudFront |

---

## 4. 패킷 흐름 (HTTP request)

```
Application (L7): GET /index.html HTTP/1.1\r\nHost:...
                              │
Presentation/Session (L6/5):  │  TLS wrap
                              │
Transport (L4):  TCP header (src/dst port, seq, ack) + payload
                              │
Network (L3):    IP header (src/dst IP) + TCP segment
                              │
Data link (L2):  Ethernet (src/dst MAC) + IP packet
                              │
Physical (L1):   bit stream
```

각 layer 가 encapsulate (다음 layer 의 payload 가 됨).

---

## 5. TCP vs UDP (L4)

| | TCP | UDP |
| --- | --- | --- |
| connection | 3-way handshake | 없음 |
| reliability | 보장 (재전송) | 없음 (best effort) |
| ordering | 보장 | 없음 |
| congestion control | 있음 | 없음 |
| 비용 | 높음 (header 20B, handshake) | 낮음 (header 8B) |
| 사용 | HTTP, SSH, FTP | DNS, NTP, VoIP, QUIC, 게임 |

→ HTTP/3 = QUIC = UDP 위 (TCP 의 단점 보완).

---

## 6. port 와 socket

```
socket = IP + port + protocol
       = 10.0.0.5:8080 (TCP)

well-known ports (0-1023): root 권한 필요
ephemeral ports (32768-60999): client 의 random
```

→ Linux 의 `net.ipv4.ip_local_port_range` 가 ephemeral 범위.

---

## 7. 연결 상태 (TCP)

```
LISTEN → SYN_SENT → SYN_RECV → ESTABLISHED → ... → FIN_WAIT → TIME_WAIT → CLOSED
```

- **TIME_WAIT** = 4분 (2 MSL) — 같은 4-tuple 재사용 방지.
- 많으면 ephemeral port 고갈.
- `net.ipv4.tcp_tw_reuse = 1`.

---

## 8. NAT (Network Address Translation)

```
[private 10.0.0.5] → [NAT (gateway 1.2.3.4)] → [internet]
    src=10.0.0.5      src 변환됨            src=1.2.3.4
                      → 같은 port 여러 client 매핑
```

→ container, VPC, 가정 router 모두 NAT.

---

## 9. CIDR / 서브넷

```
10.0.0.0/24 = 10.0.0.0 ~ 10.0.0.255 (256 IP, 254 사용 가능)
10.0.0.0/16 = 65,536 IP
10.0.0.0/8  = 16,777,216 IP

private 범위:
  10.0.0.0/8
  172.16.0.0/12
  192.168.0.0/16
```

→ VPC 디자인의 기본.

---

## 10. 관련

- [[networking-ops|↑ networking-ops]]
- [[load-balancer-types]]
- [[dns]]
- [[../linux/networking-basics|↗ linux networking]]
