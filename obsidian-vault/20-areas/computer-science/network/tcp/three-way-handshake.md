---
title: "3-Way Handshake (TCP 연결 수립)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T17:30:00+09:00
tags:
  - network
  - tcp
  - handshake
  - syn-flood
---

# 3-Way Handshake (TCP 연결 수립)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | SYN/SYN-ACK/ACK / Half-open / SYN flood / SYN cookie / TFO |

**[[tcp|↑ TCP]]** · **[[../network|↑↑ network hub]]**

---

## 1. 왜 3 번인가

TCP 의 연결 = **양쪽이 서로의 ISN (Initial Sequence Number) 을 동기화**.

```
A → B: "내 ISN은 x" (SYN)
B → A: "받았어, 내 ISN은 y" (SYN+ACK)
A → B: "받았어" (ACK)
```

2 번이면 A 의 ISN 은 B 가 알지만, B 의 ISN 을 A 가 받았는지 확인 X.
3 번이 **양방향 모두 알았다** 의 최소.

→ 동기화 + 양방향 확인 = 3 메시지가 최소.

---

## 2. 흐름

```
Client                                   Server
  |                                         |
  |-------- SYN, Seq=1000 ----------------->|
  |       (Flags=SYN, Seq=1000, ACK=0)      |
  |                                         |  LISTEN
  |                                         |     ↓
  |                                         |  SYN_RECEIVED
  |<------- SYN, ACK, Seq=2000, ACK=1001 ---|
  |       (Flags=SYN+ACK, Seq=2000, ACK=1001)|
  |  SYN_SENT                               |
  |     ↓                                   |
  |  ESTABLISHED                            |
  |-------- ACK, Seq=1001, ACK=2001 ------->|
  |       (Flags=ACK, Seq=1001, ACK=2001)   |
  |                                         |  ESTABLISHED
  |                                         |
  |== 양방향 데이터 송수신 가능 ===========|
```

### 2.1 SYN
- Client → Server
- `SYN=1, Seq=x` (x = client ISN, 랜덤)
- "연결하고 싶다, 내 첫 byte 번호는 x"

### 2.2 SYN+ACK
- Server → Client
- `SYN=1, ACK=1, Seq=y, ACK=x+1`
- "OK, 내 ISN 은 y, 너의 x+1 부터 기다린다"

### 2.3 ACK
- Client → Server
- `ACK=1, Seq=x+1, ACK=y+1`
- "OK, 시작"

→ 3 번째 ACK 부터는 데이터 같이 보낼 수 있음 (piggyback).

---

## 3. SYN 도 1 byte 처럼 — Seq 가 +1

SYN 자체는 데이터 없지만 Seq# 1 칸 차지:
- A: Seq=1000 (SYN)
- A: Seq=1001 (첫 데이터 byte)

FIN 도 동일.

---

## 4. Half-Open Connection

3 번째 ACK 이전 = **Half-Open**.
- Client: ESTABLISHED 라 생각
- Server: SYN_RECEIVED 에서 대기

문제: Server 가 자원 할당했는데 Client 가 연결 안 이어가면 → 자원 누수.

---

## 5. SYN Flood Attack

### 5.1 공격

```
공격자 → SYN, Spoofed Src IP (1.2.3.4)
공격자 → SYN, Spoofed Src IP (5.6.7.8)
공격자 → SYN, Spoofed Src IP (9.10.11.12)
...

서버: 매 SYN 에 대해
- SYN_RECEIVED 상태 진입
- TCB (Transmission Control Block) 할당
- SYN+ACK 응답 → spoofed IP 로 가서 사라짐
- 타임아웃까지 자원 유지

→ TCB 풀 소진 → 정상 사용자 SYN 받지 못함
```

### 5.2 방어 — SYN Cookie

```
1. SYN 받음 → Seq 계산:
   y = HASH(src IP, src port, dst IP, dst port, secret) + 시간 정보
2. SYN+ACK Seq=y 송신
3. TCB 할당 X! (자원 절약)
4. ACK 받으면 → ACK# - 1 = y 를 다시 HASH 와 비교
5. 일치하면 연결 수립

→ 서버 자원 절약, 정상 클라이언트만 통과
```

### 5.3 SYN Cookie 한계
- MSS 등 Options 일부 손실
- 매우 큰 부하에서만 활성 (sysctl)

### 5.4 다른 방어
- **SYN proxy** — 방화벽이 대신 handshake
- **Rate limiting** — IP 별 SYN 수 제한
- **First SYN drop** — 진짜 클라이언트는 재시도

---

## 6. RFC 793 의 "Simultaneous Open"

```
A → B: SYN
B → A: SYN  (양쪽이 동시에)
A → B: SYN+ACK
B → A: SYN+ACK
→ ESTABLISHED
```

4 메시지로 수립. 거의 발생 X (P2P 일부).

---

## 7. SYN Backlog Queue

서버의 SYN_RECEIVED 대기열:

```
Linux:
  net.core.somaxconn = 4096          # listen queue
  net.ipv4.tcp_max_syn_backlog = 8192 # SYN queue
  net.ipv4.tcp_syncookies = 1         # SYN cookie 활성
```

### Half-open queue vs Accept queue
- **Half-open** — SYN_RECEIVED 상태
- **Accept queue** — ESTABLISHED + 응용 `accept()` 대기
- 둘 다 가득 차면 SYN drop

---

## 8. RTT 측정 — 첫 RTT

```
T0: SYN 송신
T1: SYN+ACK 수신    → RTT = T1 - T0
T2: ACK 송신
```

이 첫 RTT 가 후속 RTO 계산의 시작점.

---

## 9. SYN 옵션 — 협상

SYN / SYN+ACK 에 옵션 들어감 (이후엔 들어가도 무시 가능):

| Option | 협상 |
| --- | --- |
| **MSS** | 양쪽이 자기 MSS 알림, 작은 쪽 채택 |
| **Window Scale** | 양쪽 모두 지원해야 활성 |
| **SACK Permitted** | 양쪽 모두 지원해야 SACK 사용 |
| **Timestamp** | 양쪽 모두 지원해야 |
| **TCP Fast Open** | 양쪽 모두 지원해야 |

---

## 10. TCP Fast Open (TFO, RFC 7413)

### 10.1 동기

3-way handshake = 1 RTT 낭비. 짧은 연결 (HTTP GET) 에 치명적.

### 10.2 흐름

```
첫 연결:
  C → S: SYN
  S → C: SYN+ACK + TFO Cookie
  C → S: ACK
  ...

두 번째 연결 (같은 서버):
  C → S: SYN + TFO Cookie + DATA  ← 즉시 데이터!
  S → C: SYN+ACK + DATA            ← 즉시 응답!
  C → S: ACK
```

→ **0-RTT 데이터 송수신** (재방문 시).

### 10.3 함정
- 멱등 (idempotent) 요청만 — POST 등 X (재전송 위험)
- 첫 방문은 여전히 1 RTT
- 일부 미들박스가 cookie 제거

### 10.4 QUIC 가 더 잘함
QUIC 는 표준으로 0-RTT 지원. TFO 는 사실상 사장.

---

## 11. 비정상 종료 — RST

3-way handshake 도중 RST 받으면:
- 즉시 ESTABLISHED 가 아닌 CLOSED
- 보통 "Connection refused" (포트 close) 또는 "RST attack"

### 11.1 포트 close 시
```
C → S: SYN (포트 12345)
S → C: RST  ← 그 포트 안 열림
```

### 11.2 RST 공격
- Sequence# 추측 후 RST 주입 → 연결 강제 종료
- 1985 Morris 공격의 기반
- 모던: ISN 랜덤화 + 윈도우 안의 RST 만 수용

---

## 12. tcpdump 로 보기

```bash
sudo tcpdump -i en0 -nn 'tcp[tcpflags] & (tcp-syn|tcp-ack) != 0'

# 출력:
# 12:00:01.123 IP A.49152 > B.80: Flags [S], seq 1000
# 12:00:01.234 IP B.80 > A.49152: Flags [S.], seq 2000, ack 1001
# 12:00:01.235 IP A.49152 > B.80: Flags [.], ack 2001
```

`[S]` SYN, `[S.]` SYN+ACK, `[.]` ACK (`[F]` FIN, `[R]` RST, `[P]` PSH).

---

## 13. 함정

### 함정 1 — Backlog 부족
서버 SYN queue 가득 → SYN drop → 클라 "connect timeout".

### 함정 2 — SYN Cookie 활성 후 옵션 손실
Window Scale / SACK 잃음 → 성능 ↓ (보안 vs 성능 트레이드오프).

### 함정 3 — 방화벽 / NAT 의 SYN 추적
NAT 가 SYN 만 보고 흐름 추적 — 끊긴 연결 인식 못 함.

### 함정 4 — Backlog 작은 값
`somaxconn=128` 기본은 너무 작음. 1024+ 권장.

### 함정 5 — TCP Fast Open 의 호환성
미들박스가 옵션 제거 → 일반 handshake 로 폴백. 측정 시 차이 작음.

---

## 14. 학습 자료

- RFC 793 (Section 3.4), RFC 9293
- RFC 4987 (SYN flood 방어), RFC 7413 (TFO)
- **TCP/IP Illustrated Vol.1** Ch. 18
- Linux Kernel `net/ipv4/tcp_input.c`

---

## 15. 관련

- [[four-way-termination]] — 종료
- [[tcp-state-machine]] — 상태 전이
- [[tcp-header]] — SYN/ACK flag
- [[tcp-attacks]] — SYN flood 깊이
- [[tcp]] — TCP hub
