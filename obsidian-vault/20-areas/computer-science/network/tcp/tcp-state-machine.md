---
title: "TCP 상태 머신 (State Machine)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T17:40:00+09:00
tags:
  - network
  - tcp
  - state-machine
---

# TCP 상태 머신 (State Machine)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 11 상태 + 전이 + ss 출력 |

**[[tcp|↑ TCP]]** · **[[../network|↑↑ network hub]]**

---

## 1. 11 가지 상태

| 상태 | 의미 |
| --- | --- |
| **CLOSED** | 연결 없음 |
| **LISTEN** | 서버 — 연결 대기 |
| **SYN_SENT** | 클라 — SYN 보낸 후 SYN+ACK 대기 |
| **SYN_RECEIVED** | 서버 — SYN 받고 SYN+ACK 보낸 후 ACK 대기 |
| **ESTABLISHED** | 데이터 송수신 가능 |
| **FIN_WAIT_1** | Active close — FIN 보냄, ACK 대기 |
| **FIN_WAIT_2** | Active close — FIN 의 ACK 받음, 상대 FIN 대기 |
| **CLOSE_WAIT** | Passive close — FIN 받고 ACK 보냄, 응용 close 대기 |
| **CLOSING** | 양쪽 동시 close — drop ACK 대기 |
| **LAST_ACK** | Passive close — FIN 보낸 후 마지막 ACK 대기 |
| **TIME_WAIT** | Active close — 2×MSL 대기 (옛 패킷 사라지기 기다림) |

---

## 2. 전이 다이어그램

```
                              CLOSED
                              /  ↑\
              passive open   /    \  active open
                            /      \  → SYN
                          (CLOSE)    SYN_SENT ─→ (CLOSE/timeout)
                            /         |              ↓
                       LISTEN   ←─── SYN_RECEIVED  CLOSED
                          ↓           ↑   ↓
                  rcv SYN →           |   |
                  snd SYN+ACK         |   |
                          ↓           |   |
                  ESTABLISHED ────────┘   |
                       ↓                  |
                       ┌───────────────────
                       │ rcv FIN              │ active close
                       │ snd ACK              ↓
                       ↓ (Passive close)   FIN_WAIT_1 ──(rcv FIN+ACK,
                  CLOSE_WAIT                  ↓                snd ACK)
                       ↓ close()         (rcv ACK)            ↓
                  LAST_ACK            FIN_WAIT_2 → TIME_WAIT
                       ↓ rcv ACK            ↓ rcv FIN, snd ACK
                       ↓                    ↓
                  CLOSED              TIME_WAIT
                                            ↓ 2×MSL
                                          CLOSED
```

(텍스트 한계로 단순화 — RFC 793 의 그림 참조)

---

## 3. 상태별 동작

### 3.1 CLOSED
- 가상 상태 — 연결 없음
- `socket()` 후 `bind()` 전

### 3.2 LISTEN
- `listen()` 호출 후
- SYN 받기 대기
- 자원: SYN backlog queue + accept queue

### 3.3 SYN_SENT
- 클라이언트가 `connect()` 호출
- SYN 송신 후
- SYN+ACK 받으면 → ESTABLISHED 로 전이
- 타임아웃 시 → CLOSED

### 3.4 SYN_RECEIVED
- 서버가 SYN 받고 SYN+ACK 송신
- ACK 받으면 → ESTABLISHED
- 자원: TCB (Transmission Control Block) 할당

### 3.5 ESTABLISHED
- **데이터 송수신 가능**
- 가장 시간 오래 머무는 상태
- `read()`, `write()` 동작

### 3.6 FIN_WAIT_1
- Active close 가 `close()` 호출
- FIN 송신 직후
- ACK 또는 FIN+ACK 받기 대기
- 양쪽 동시 close 면 → CLOSING

### 3.7 FIN_WAIT_2
- FIN_WAIT_1 에서 ACK 받음
- 상대의 FIN 대기
- Linux 기본 60 초 타임아웃 (`tcp_fin_timeout`)

### 3.8 CLOSE_WAIT
- Passive close — FIN 받고 ACK 보냄
- 응용이 `close()` 호출 대기
- **응용 버그 시 무한** — 누수 위험

### 3.9 CLOSING
- 양쪽이 동시 FIN
- 자기 FIN 의 ACK 대기
- 드문 상태

### 3.10 LAST_ACK
- Passive close 가 자기 FIN 보냄
- 마지막 ACK 받기 대기
- 받으면 → CLOSED

### 3.11 TIME_WAIT
- Active close 의 마지막 상태
- 2 × MSL (보통 60s - 2min) 대기
- 옛 패킷 정리 + ACK 손실 대비

자세히 → [[four-way-termination#5 TIME_WAIT]]

---

## 4. 정상 전이 사이클

### 4.1 클라이언트 (Active)
```
CLOSED → SYN_SENT → ESTABLISHED →
  FIN_WAIT_1 → FIN_WAIT_2 → TIME_WAIT → CLOSED
```

### 4.2 서버 (Passive Open + Passive Close)
```
CLOSED → LISTEN → SYN_RECEIVED → ESTABLISHED →
  CLOSE_WAIT → LAST_ACK → CLOSED
```

### 4.3 서버 (Passive Open + Active Close)
```
CLOSED → LISTEN → SYN_RECEIVED → ESTABLISHED →
  FIN_WAIT_1 → FIN_WAIT_2 → TIME_WAIT → CLOSED
```

---

## 5. 비정상 전이

### RST 수신
- 모든 상태 → 즉시 CLOSED
- 응용에 "Connection reset by peer"

### 타임아웃
- SYN_SENT 75 초 → CLOSED ("Connection timed out")
- FIN_WAIT_2 60 초 → CLOSED
- TIME_WAIT 60-240 초 → CLOSED

---

## 6. ss / netstat 로 상태 확인

```bash
# Linux 모던
ss -tan
# 출력:
# State       Recv-Q Send-Q Local Address:Port    Peer Address:Port
# LISTEN      0      128    0.0.0.0:22            0.0.0.0:*
# ESTAB       0      0      192.168.1.5:48562     93.184.216.34:443
# TIME-WAIT   0      0      192.168.1.5:48230     93.184.216.34:443

# 상태별 카운트
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c
#  10 ESTAB
#   1 LISTEN
#   25 TIME-WAIT

# 특정 상태만
ss -tan state established
ss -tan state time-wait
ss -tan state close-wait

# 옛 netstat
netstat -an | grep TIME_WAIT | wc -l
```

---

## 7. 진단 패턴

### 패턴 1 — 대량 TIME_WAIT
**증상**: 수천 ~ 수만 TIME_WAIT
**원인**: 짧은 연결 폭주 (Active Close)
**해결**: Connection pool, Keep-alive, SO_REUSEADDR

### 패턴 2 — 대량 CLOSE_WAIT
**증상**: CLOSE_WAIT 누적
**원인**: 응용 `close()` 누락 (버그)
**해결**: 코드 수정 — try-finally, defer, with

### 패턴 3 — 대량 FIN_WAIT_2
**증상**: FIN_WAIT_2 누적
**원인**: 상대 응용이 close 안 함
**해결**: `tcp_fin_timeout` 단축 (60→30), 상대 응용 수정

### 패턴 4 — SYN_RECEIVED 폭증
**증상**: SYN_RECEIVED 다수
**원인**: **SYN flood 공격**
**해결**: SYN cookie, Rate limit, 방화벽

### 패턴 5 — LISTEN backlog 가득
**증상**: 새 연결 거부
**원인**: 응용이 `accept()` 빨리 안 함 / `somaxconn` 작음
**해결**: `accept()` 빠르게, backlog 늘림

---

## 8. Linux sysctl 관련

```bash
# 상태 타임아웃
net.ipv4.tcp_fin_timeout = 60          # FIN_WAIT_2

# TIME_WAIT 재사용
net.ipv4.tcp_tw_reuse = 1              # 송신만
# net.ipv4.tcp_tw_recycle = 0          # 제거됨 (4.12+)

# Listen backlog
net.core.somaxconn = 4096

# SYN backlog
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_syncookies = 1

# Keepalive
net.ipv4.tcp_keepalive_time = 7200     # 2 시간
net.ipv4.tcp_keepalive_intvl = 75
net.ipv4.tcp_keepalive_probes = 9
```

---

## 9. 함정

### 함정 1 — TIME_WAIT 짧게 만들기
2×MSL 의 의미 무시 — 데이터 손실 가능.

### 함정 2 — CLOSE_WAIT 의 원인 OS 가정
거의 항상 응용 버그. OS 튜닝이 아닌 코드 수정.

### 함정 3 — listen() 직후 accept 안 함
backlog 가득 → 새 SYN drop.

### 함정 4 — RST 의 의미
"Connection reset" — 상대가 의도적 또는 응용 크래시. 분석 필요.

### 함정 5 — Half-open 무시
SYN_RECEIVED 누적 → SYN flood 의심.

---

## 10. 학습 자료

- RFC 793 Section 3.2, RFC 9293
- **TCP/IP Illustrated Vol.1** Appendix B (State Diagram)
- Linux Kernel `tcp.c` `tcp_set_state()`
- "Coping with TIME_WAIT" — Vincent Bernat

---

## 11. 관련

- [[three-way-handshake]] — SYN_SENT / SYN_RECEIVED
- [[four-way-termination]] — FIN_WAIT_1/2 / CLOSE_WAIT / TIME_WAIT
- [[tcp]] — TCP hub
