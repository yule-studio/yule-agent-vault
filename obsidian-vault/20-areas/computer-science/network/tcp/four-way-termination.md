---
title: "4-Way Termination (TCP 연결 종료)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T17:35:00+09:00
tags:
  - network
  - tcp
  - termination
  - time-wait
---

# 4-Way Termination (TCP 연결 종료)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | FIN/ACK/FIN/ACK + TIME_WAIT + Half-close + SO_REUSEADDR |

**[[tcp|↑ TCP]]** · **[[../network|↑↑ network hub]]**

---

## 1. 왜 4 번인가

TCP 는 **양방향 독립** — 각 방향이 따로 종료.

```
A → B: "내 송신 끝" (FIN)
B → A: "OK" (ACK)
B → A: (이후 데이터 더 보낼 수 있음)
B → A: "내 송신도 끝" (FIN)
A → B: "OK" (ACK)
```

3-way handshake 처럼 ACK 가 합쳐질 수도 있지만 보통 분리 (B 가 보낼 데이터 있어서).

---

## 2. 흐름

```
Client (Active Close)                       Server (Passive Close)
  | ESTABLISHED                                ESTABLISHED |
  |                                                        |
  |-------- FIN, Seq=u ------------------------------>     |
  |  FIN_WAIT_1                              CLOSE_WAIT    |
  |<------- ACK, Seq=v, ACK=u+1 ----------------------     |
  |  FIN_WAIT_2                                            |
  |                                                        |
  |          (B 가 남은 데이터 보낼 수 있음)              |
  |          (이 동안 A 는 받기만 가능)                    |
  |                                                        |
  |<------- FIN, ACK, Seq=v, ACK=u+1 -----------------     |
  |  TIME_WAIT                               LAST_ACK      |
  |-------- ACK, Seq=u+1, ACK=v+1 ------------------>      |
  |                                          CLOSED        |
  |                                                        |
  |  (2 × MSL 대기)                                        |
  |     ↓                                                  |
  |  CLOSED                                                |
```

---

## 3. 각 메시지

### 3.1 FIN (Client → Server)
- Client 가 "더 보낼 데이터 없음" 알림
- `FIN=1, Seq=u`
- Client 는 `close()` 호출

### 3.2 ACK (Server → Client)
- Server 가 FIN 받음 확인
- `ACK=1, Seq=v, ACK=u+1`
- 응용은 아직 `read()` 가 EOF 받음

### 3.3 FIN (Server → Client)
- Server 도 `close()` 호출
- `FIN=1, ACK=1, Seq=w, ACK=u+1`
- (마지막 ACK 와 합쳐 보내기도)

### 3.4 ACK (Client → Server)
- Client 가 마지막 ACK
- `ACK=1, Seq=u+1, ACK=w+1`
- Client: TIME_WAIT 진입

---

## 4. 능동적 / 수동적 종료

| 종류 | 누가 |
| --- | --- |
| **Active Close** | 먼저 FIN 보낸 쪽 |
| **Passive Close** | FIN 받고 응답하는 쪽 |

→ Active Close 측이 **TIME_WAIT** 진입. Passive 는 즉시 CLOSED.

보통 클라이언트가 Active 지만, 서버 (응답 후 connection close) 일 수도.

---

## 5. TIME_WAIT — 가장 중요한 상태

### 5.1 정의
Active Close 측이 마지막 ACK 보낸 후 **2 × MSL** 동안 대기.

MSL (Maximum Segment Lifetime) = 패킷의 최대 수명. RFC 1122 권장 2 분 → 보통 30 초 - 2 분.

→ 2 × MSL = **60 초 - 4 분** 보통.

### 5.2 왜 필요한가

#### 이유 1 — 마지막 ACK 손실 대비

```
A → B: FIN (응답으로 ACK 보낼 차례)
A → B: ACK (마지막)  ← 손실
B → A: FIN (B 가 재전송)
A → B: ACK   ← 다시
```

A 가 즉시 CLOSED 되면 B 의 FIN 재전송에 RST 응답 → 비정상.

#### 이유 2 — 옛 패킷 방지 (Connection muddling)

```
같은 (src IP/port, dst IP/port) 4-tuple 의 두 연결이 연달아 일어나면
첫 연결의 늦은 패킷이 두 번째 연결에 섞임
```

2 × MSL 대기로 모든 옛 패킷이 망에서 사라지길 보장.

### 5.3 부작용

- **포트 재사용 불가** — 같은 (src/dst, port) 못 씀
- **부하 큰 서버에서 소켓 누적** — 메모리 / 포트 소진

```bash
# Linux 에서 TIME_WAIT 갯수
ss -an state time-wait | wc -l
netstat -an | grep TIME_WAIT | wc -l
```

### 5.4 해결 / 완화

#### SO_REUSEADDR
- bind() 시 TIME_WAIT 의 같은 주소 / 포트 재사용 허용
- 서버에서 흔히

#### SO_REUSEPORT (Linux 3.9+)
- 여러 소켓이 같은 포트 binding
- 부하 분산 (nginx worker)

#### sysctl
```bash
# 절대 X — 안전성 깨짐
net.ipv4.tcp_tw_recycle = 0  # 제거됨 (Linux 4.12+)

# 재사용 (송신만)
net.ipv4.tcp_tw_reuse = 1

# MSL 단축 (조심)
net.ipv4.tcp_fin_timeout = 60  # FIN_WAIT_2 타임아웃

# 포트 범위 늘리기
net.ipv4.ip_local_port_range = "10240 65535"
```

#### Connection Pool
- HTTP keep-alive
- DB connection pool
- 연결 재사용 → TIME_WAIT 감소

---

## 6. CLOSE_WAIT — 서버 측 누수

```
서버: FIN 받고 → ACK 보냄 → CLOSE_WAIT
       응용이 close() 호출 안 함  ← !!!
       무한 CLOSE_WAIT
```

### 6.1 증상
- `netstat | grep CLOSE_WAIT` 가 폭증
- 파일 디스크립터 누수

### 6.2 원인 — 응용 버그
- 응용이 read() EOF 를 무시
- close() 호출 누락
- 예외 처리에서 close 안 함

### 6.3 해결
- 응용 코드 수정
- try-with-resources / defer / context manager

---

## 7. Half-Close (양방향 독립 종료)

```
shutdown(sock, SHUT_WR);   # 송신만 종료, 수신 계속
shutdown(sock, SHUT_RD);   # 수신만 종료
shutdown(sock, SHUT_RDWR); # 양방향 (= close)
```

### 7.1 사용
- 클라이언트가 요청 끝 알리고 응답 기다림
- HTTP 의 client → server EOF 후 응답 받기 (안 흔함)

### 7.2 FIN_WAIT_2 의 위험
- Half-close 후 상대가 close 안 함 → FIN_WAIT_2 무한
- 모던 OS 는 타임아웃 (Linux 기본 60s)

---

## 8. RST (Abortive Close)

정상 4-way 대신 즉시 RST 보내기:

```c
struct linger lo = {1, 0};   // l_onoff=1, l_linger=0
setsockopt(sock, SOL_SOCKET, SO_LINGER, &lo, sizeof(lo));
close(sock);   // → RST 송신, TIME_WAIT X
```

### 8.1 사용
- 비정상 종료 알리기
- TIME_WAIT 피하기 (위험)
- DDoS 대응

### 8.2 위험
- 버퍼의 데이터 손실
- 신뢰성 깨짐

---

## 9. 동시 Close (Simultaneous Close)

```
A → B: FIN
B → A: FIN  (이미 동시에 보냄)
A: CLOSING (ACK 대기)
B: CLOSING
A → B: ACK
B → A: ACK
→ TIME_WAIT × 양쪽
```

P2P 등 드물게 발생.

---

## 10. 상태 전이 (Active Close)

```
ESTABLISHED
   ↓ close()
FIN_WAIT_1
   ↓ FIN ACK 받음
FIN_WAIT_2
   ↓ FIN 받음 + ACK 송신
TIME_WAIT
   ↓ 2 × MSL 대기
CLOSED
```

자세한 상태 머신 → [[tcp-state-machine]]

---

## 11. 함정

### 함정 1 — TIME_WAIT 대량 누적
짧은 연결 폭주 → 포트 소진. Connection pool / Keep-alive.

### 함정 2 — CLOSE_WAIT 누수
응용 close() 누락. 코드 리뷰 / try-with-resources.

### 함정 3 — `tcp_tw_recycle` 사용
NAT 환경에서 다른 호스트의 패킷 폐기 → 통신 불가. **절대 사용 X**.

### 함정 4 — Half-close 후 read 안 함
FIN_WAIT_2 무한.

### 함정 5 — RST 폐기
`SO_LINGER 0` 는 데이터 손실 위험. 의도적으로만.

### 함정 6 — TIME_WAIT 가 클라이언트에도
클라이언트가 먼저 close 하면 클라 측 TIME_WAIT. ephemeral 포트 소진 가능.

---

## 12. 디버깅

```bash
# 상태별 카운트
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c

# TIME_WAIT 만
ss -tan state time-wait

# CLOSE_WAIT — 응용 버그
ss -tan state close-wait

# FIN_WAIT_2 — 상대가 close 안 함
ss -tan state fin-wait-2
```

```bash
# tcpdump 로 FIN 확인
sudo tcpdump -i en0 -nn 'tcp[tcpflags] & tcp-fin != 0'
```

---

## 13. 학습 자료

- RFC 793 Section 3.5, RFC 1122, RFC 9293
- **TCP/IP Illustrated Vol.1** Ch. 18.6, 19
- "Coping with the TCP TIME-WAIT state on busy Linux servers" — Vincent Bernat
- Linux Kernel `tcp_close()` in `tcp.c`

---

## 14. 관련

- [[three-way-handshake]] — 연결 수립
- [[tcp-state-machine]] — 상태 머신
- [[tcp]] — TCP hub
