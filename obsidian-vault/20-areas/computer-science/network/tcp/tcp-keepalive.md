---
title: "TCP Keepalive & Half-Open Detection"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T18:00:00+09:00
tags:
  - network
  - tcp
  - keepalive
---

# TCP Keepalive & Half-Open Detection

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Keepalive 동작 / sysctl / 응용 keepalive |

**[[tcp|↑ TCP]]** · **[[../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

ESTABLISHED 연결의 **상대가 살아있는지 주기적 확인**. RFC 1122 권장,
기본 비활성 (응용이 활성화).

---

## 2. 왜 필요한가

### 2.1 Half-Open Connection

```
A ───── ESTABLISHED ───── B
         (정상)
   ↓
B 크래시 / 전원 OFF / 네트워크 단절
   ↓
A 는 여전히 ESTABLISHED  ← 모름
A 는 데이터 보내봐야 응답 X
```

→ **TCP 자체는 연결 끊김 감지 X** — 데이터 안 보내면 영원히 상태 유지.

### 2.2 누가 알게 되는가
- A 가 새 데이터 보냄 → 재전송 → RTO timeout (수 분) → 마침내 ERROR
- A 가 보낼 데이터 없으면 **영원히** 미감지

### 2.3 Keepalive 의 역할
- 주기적 작은 패킷 송신 (1 byte 데이터, 옛 ACK#)
- 응답 없으면 → 연결 끊김 판단

---

## 3. Keepalive 동작 (RFC 1122)

```
1. Idle 시간 측정 (data 없는 시간)
2. tcp_keepalive_time 도달 → Keepalive Probe 송신
3. 응답 (ACK) 받으면 → 다시 idle 측정 시작
4. 응답 없으면 → tcp_keepalive_intvl 후 재시도
5. tcp_keepalive_probes 회 실패 → 연결 종료
```

### 3.1 Keepalive Probe 내용

```
ACK#: 마지막 ACK + 1 (잘못된 ACK# — 의도적)
데이터: 1 byte (또는 0 byte, 구현마다)
```

상대는 `ACK#` 잘못된 거 확인 → 정상 ACK 응답. (실제로는 OS 가 자동 처리.)

---

## 4. Linux sysctl

```bash
# 기본 (Linux)
net.ipv4.tcp_keepalive_time = 7200      # 2 시간 idle 후 첫 probe
net.ipv4.tcp_keepalive_intvl = 75       # probe 간격 75 초
net.ipv4.tcp_keepalive_probes = 9       # 9 번 실패 시 종료
```

→ 기본 2 시간 + 9 × 75초 = 약 **2 시간 11 분** 까지 stuck 후 종료.

### 4.1 단축 권장

```bash
net.ipv4.tcp_keepalive_time = 600       # 10 분
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 3
# → 10 분 + 3 × 30 = 약 11 분
```

LB / 데이터센터 환경에선 5-10 분이 일반적.

---

## 5. 응용 코드 — Keepalive 활성

### 5.1 C (POSIX)

```c
int yes = 1;
setsockopt(sock, SOL_SOCKET, SO_KEEPALIVE, &yes, sizeof(yes));

// Linux 만 — per-socket 설정
int idle = 60;        // 60 초 idle 후 시작
int intvl = 10;       // 10 초 간격
int cnt = 5;          // 5 번 실패 시 종료
setsockopt(sock, IPPROTO_TCP, TCP_KEEPIDLE, &idle, sizeof(idle));
setsockopt(sock, IPPROTO_TCP, TCP_KEEPINTVL, &intvl, sizeof(intvl));
setsockopt(sock, IPPROTO_TCP, TCP_KEEPCNT, &cnt, sizeof(cnt));
```

### 5.2 Python

```python
import socket
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.setsockopt(socket.SOL_SOCKET, socket.SO_KEEPALIVE, 1)
# Linux 만
sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPIDLE, 60)
sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPINTVL, 10)
sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPCNT, 5)
```

### 5.3 Go

```go
import "net"
conn, _ := net.Dial("tcp", "host:port")
tcpConn := conn.(*net.TCPConn)
tcpConn.SetKeepAlive(true)
tcpConn.SetKeepAlivePeriod(30 * time.Second)
```

---

## 6. NAT / 방화벽 의 영향

### 6.1 NAT 의 timeout

- NAT 가 흐름 추적
- 데이터 없으면 일정 시간 후 entry 삭제 (보통 60초 - 5분)
- → 다시 데이터 보낼 때 NAT 가 모름 → RST 또는 폐기

### 6.2 Keepalive 가 해결
- NAT timeout 보다 짧은 주기로 keepalive
- 흐름 유지

→ 모바일 / Cellular NAT 환경에선 30-60 초 keepalive 흔함.

### 6.3 HTTP/2 PING / WebSocket PING
- 응용 레벨 keepalive
- TCP keepalive 보다 정확 (응용 살아있음 확인)

---

## 7. 응용 레벨 vs TCP Keepalive

| 측면 | TCP Keepalive | 응용 Keepalive |
| --- | --- | --- |
| 위치 | OS | 응용 |
| 감지 | 네트워크 단절 | 응용 행 |
| 비용 | 작음 | 작음 |
| 정확도 | TCP 만 | 응용 살아있음 |
| 협상 | OS 옵션 | 프로토콜 정의 |

### 응용 Keepalive 예
- **HTTP/2** — PING frame
- **WebSocket** — Ping/Pong
- **gRPC** — Keepalive
- **Redis** — `tcp-keepalive` config
- **MQTT** — Keepalive Interval

→ 보통 둘 다 활성.

---

## 8. Keepalive 의 함정

### 함정 1 — 기본값 너무 김
2 시간 → 11 분 변경 권장.

### 함정 2 — TCP Keepalive 가 응용 행 감지 X
응용은 살아있는데 stuck (deadlock) — TCP 는 정상 → Keepalive 응답 OK.
→ 응용 레벨 health check 필요.

### 함정 3 — 미들박스의 작은 timeout
방화벽 / NAT 가 60 초 timeout → Keepalive 가 30 초 권장.

### 함정 4 — DDoS 표적
대량 idle 연결 + Keepalive → 자원 소진. 정책 / 제한.

### 함정 5 — Keepalive 의 데이터 사용 부담
모바일 데이터 셈 (드물지만 미세). 너무 짧은 주기 X.

---

## 9. 학습 자료

- RFC 1122 Section 4.2.3.6
- Linux Kernel `tcp_send_probe0()`
- "TCP Keepalive HOWTO" — TLDP
- 응용별 keepalive — HTTP/2 RFC 7540, WebSocket RFC 6455

---

## 10. 관련

- [[four-way-termination]] — TIME_WAIT
- [[tcp-state-machine]] — ESTABLISHED 의 의미
- [[tcp]] — TCP hub
