---
title: "L4 — 전송 계층 (Transport Layer)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T17:00:00+09:00
tags:
  - network
  - osi
  - layer-4
  - tcp
  - udp
---

# L4 — 전송 계층 (Transport Layer)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | L4 hub + 별도 폴더 (tcp/ udp/ quic/) 안내 |

**[[../osi-7-layer|↑ OSI 7 계층]]** · **[[../../network|↑↑ network hub]]**

> L4 의 깊이는 [[../../tcp/tcp]] / [[../../udp/udp]] / [[../../quic/quic]] 폴더 참조.
> 이 노트는 L4 의 일반 개념 / 포트 / 비교.

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **전송 단위** | **Segment** (TCP) / **Datagram** (UDP) |
| **주소 / 식별자** | **Port** (16 bit) |
| **대표 장치** | L4 Switch (NAT, LB) |
| **대표 프로토콜** | **TCP**, **UDP**, **QUIC**, SCTP, DCCP |
| **상위 계층** | L5 / L7 Application |
| **하위 계층** | L3 Network (IP) |

---

## 1. 한 줄 정의

**End-to-end (호스트 ↔ 호스트) 데이터 전달**. 포트로 응용 식별. 신뢰성 (TCP) /
비신뢰성 (UDP) 선택. **응용에게 통신 추상화 제공**.

---

## 2. L3 (IP) 와 L4 의 차이

| 측면 | L3 (IP) | L4 (TCP/UDP) |
| --- | --- | --- |
| 식별자 | IP 주소 (호스트) | Port (응용 / 소켓) |
| 단위 | Packet | Segment / Datagram |
| 신뢰성 | 없음 (best-effort) | TCP: 있음, UDP: 없음 |
| 순서 | 보장 X | TCP: O, UDP: X |
| 흐름 제어 | 없음 | TCP: O, UDP: X |
| 혼잡 제어 | 없음 | TCP: O, UDP: X (응용 부담) |

---

## 3. Port — 포트

### 3.1 16-bit 식별자 (0-65535)

| 범위 | 분류 | 비고 |
| --- | --- | --- |
| **0-1023** | Well-known | IANA 등록, root 권한 필요 (Unix) |
| **1024-49151** | Registered | IANA 등록 (Apache, MySQL 등) |
| **49152-65535** | Dynamic / Ephemeral | 클라이언트 자동 |

### 3.2 잘 알려진 포트

| 포트 | 프로토콜 | 응용 |
| --- | --- | --- |
| 20/21 | TCP | FTP (data / control) |
| 22 | TCP | SSH |
| 23 | TCP | Telnet |
| 25 | TCP | SMTP |
| **53** | UDP/TCP | **DNS** |
| 67/68 | UDP | DHCP (server/client) |
| **80** | TCP | **HTTP** |
| 110 | TCP | POP3 |
| 119 | TCP | NNTP |
| 123 | UDP | NTP |
| 137-139 | UDP/TCP | NetBIOS |
| 143 | TCP | IMAP |
| 161/162 | UDP | SNMP |
| 179 | TCP | BGP |
| 389 | TCP | LDAP |
| **443** | TCP/UDP | **HTTPS / HTTP/3** |
| 445 | TCP | SMB |
| 465 | TCP | SMTPS |
| 514 | UDP | Syslog |
| 587 | TCP | SMTP submission |
| 636 | TCP | LDAPS |
| 993 | TCP | IMAPS |
| 995 | TCP | POP3S |
| 1433 | TCP | MSSQL |
| 3306 | TCP | MySQL |
| 3389 | TCP | RDP |
| 5432 | TCP | PostgreSQL |
| 5900 | TCP | VNC |
| **6379** | TCP | **Redis** |
| 8080 | TCP | HTTP alt |
| **8443** | TCP | HTTPS alt |
| 9000-9001 | TCP | PHP-FPM, Tor |
| 9092 | TCP | Kafka |
| 11211 | TCP | Memcached |
| 27017 | TCP | MongoDB |

### 3.3 Socket = (IP, Port, Protocol)

```
TCP 연결 = (Src IP, Src Port, Dst IP, Dst Port)  의 4-tuple
```

같은 서버 포트 80 에 여러 클라이언트 동시 연결 가능 (각자 Src Port 다름).

---

## 4. L4 프로토콜 비교

| 프로토콜 | 신뢰성 | 순서 | 흐름 / 혼잡 | 헤더 | 용도 |
| --- | --- | --- | --- | --- | --- |
| **TCP** | ✅ | ✅ | ✅ | 20+ byte | HTTP, SSH, FTP |
| **UDP** | ❌ | ❌ | ❌ | 8 byte | DNS, VoIP, 게임 |
| **QUIC** | ✅ | ✅ (스트림별) | ✅ | UDP 위 | HTTP/3 |
| **SCTP** | ✅ | 부분 | ✅ | 가변 | 전화 신호 (SS7) |
| **DCCP** | ❌ | ❌ | ✅ | 16+ byte | 거의 사장 |
| **RTP** | ❌ | ✅ | ❌ | UDP 위 | 비디오 / 오디오 |

자세히:
- [[../../tcp/tcp|↗ TCP 폴더]]
- [[../../udp/udp|↗ UDP 폴더]]
- [[../../quic/quic|↗ QUIC 폴더]]

---

## 5. NAT (Network Address Translation)

L3 의 IP 와 L4 의 Port 를 함께 다룸 (PAT / NAPT).

```
내부 (10.0.0.5:55000) → NAT → 외부 (203.0.113.1:43210)
NAT Table:
  (203.0.113.1, 43210) ↔ (10.0.0.5, 55000)
```

자세히 → [[../../ip/nat]]

---

## 6. L4 Switch / Load Balancer

L4 까지 보고 부하 분산 / NAT:
- **L4 LB** — IP + Port hash, 빠름
- 예: AWS NLB, HAProxy in TCP mode, Nginx stream

자세히 → [[../../load-balancing/load-balancing]]

---

## 7. 함정

### 함정 1 — UDP 가 무조건 빠름 가정
신뢰성 응용에서 UDP + 재구현 → 성능 ↓ (QUIC 외).

### 함정 2 — TCP 가 무조건 신뢰 가정
짧은 연결에 TCP — 핸드셰이크 비용. QUIC 0-RTT 가 대안.

### 함정 3 — Port 충돌
같은 (IP, Port) 두 응용 — 한쪽만 bind 가능.

### 함정 4 — Ephemeral Port 소진
짧은 연결 폭주 + TIME_WAIT 누적 → 포트 소진. SO_REUSEADDR, 풀.

### 함정 5 — NAT 의 L4 의존
일부 L4 (ICMP, SCTP, GRE) NAT 어려움.

---

## 8. 학습 자료

- RFC 793 (TCP), RFC 768 (UDP), RFC 9000 (QUIC)
- **TCP/IP Illustrated Vol.1** Ch. 17-24
- **UNIX Network Programming** (Stevens)
- IANA Port Number 등록

---

## 9. 관련

- [[../layer-3-network/layer-3-network]] — 하위
- [[../layer-5-session/layer-5-session]] — 상위
- [[../../tcp/tcp|↗ TCP 폴더]]
- [[../../udp/udp|↗ UDP 폴더]]
- [[../../quic/quic|↗ QUIC 폴더]]
- [[../osi-7-layer]] — OSI hub
