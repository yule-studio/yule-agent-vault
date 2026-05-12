---
title: "TCP Sliding Window & Flow Control"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T17:45:00+09:00
tags:
  - network
  - tcp
  - sliding-window
  - flow-control
---

# TCP Sliding Window & Flow Control

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 슬라이딩 윈도우 / Receive Window / Zero Window / Silly Window / SWS |

**[[tcp|↑ TCP]]** · **[[../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

TCP 의 **흐름 제어 (Flow Control)** — 송신자가 수신자의 버퍼 크기에 맞춰
보내는 양 조절. **Window 필드 (16 bit)** 가 매 ACK 마다 갱신.

---

## 2. Sliding Window 의 본질

```
송신자 측 byte stream:

| 보냄 + ACK | 보냄, ACK 대기 | 보낼 수 있음 | 못 보냄 |
              ↑                 ↑                ↑
            LastAck           NextSend         LastAck+Window

Window = 수신자가 받을 수 있는 buffer

송신 가능 = LastAck+Window 까지
```

수신자의 ACK + Window 가 도착하면 **윈도우가 오른쪽으로 슬라이드**.

---

## 3. 수신자 측

```
수신 byte stream:

| 받음 + 응용에 전달 | 받음, 응용 미수신 | 받을 수 있음 | 받을 공간 X |
                       ↑              ↑              ↑
                     NextExpected   buffer 끝      buffer 가득

Window = 받을 수 있는 byte (free buffer)
ACK = NextExpected (다음 기대 byte)
```

응용이 `read()` 호출 → buffer 비움 → window 증가.

---

## 4. Window 필드

```
TCP 헤더의 Window 필드 (16 bit) = 0 - 65535 byte
```

→ 최대 64 KB. 빠른 링크 (Long Fat Network) 에선 부족.

### Window Scale (RFC 1323)
```
실제 Window = Window 필드 × 2^ScaleFactor
ScaleFactor: 0-14
→ 최대 Window = 65535 × 2^14 = 1 GB
```

SYN / SYN+ACK 에서만 협상 (이후 변경 X).

---

## 5. BDP — Bandwidth-Delay Product

```
BDP = Bandwidth × RTT
```

링크의 "in-flight" 데이터 한계. Window 가 BDP 이상이어야 full throughput.

### 5.1 예시

- 1 Gbps + RTT 100ms (한국-미국):
  - BDP = 10⁹ × 0.1 = 100 Mbit = **12.5 MB**
  - Window 64 KB → 65535 × 8 / 0.1 = **5.2 Mbps** (실제 1 Gbps 못 채움)
  - → Window Scale 필요

### 5.2 Linux 버퍼 자동 튜닝

```bash
net.ipv4.tcp_rmem = "4096 87380 6291456"   # min default max
net.ipv4.tcp_wmem = "4096 16384 4194304"
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_moderate_rcvbuf = 1
```

---

## 6. Zero Window

수신 buffer 가득:
```
받는 측: ACK + Window=0  ← "더 보내지 마"
보내는 측: 멈춤
```

### 6.1 Zero Window Probe

송신자가 멈춰 있을 때 수신자의 Window 갱신 알림 (Window > 0) 손실 시:
- 송신자가 영원히 stuck
- → **Persist Timer** — 주기적 (5s, 15s, 30s, 1m, ...) 1-byte probe
- 수신자가 zero ACK 또는 non-zero Window 응답

### 6.2 Zero Window 의 함정

- 응용이 `read()` 안 해 buffer 가득 → 영원히 stall
- 진단: `ss -tin` 의 `wnd`

---

## 7. Silly Window Syndrome (SWS)

작은 window 갱신이 작은 segment 폭증을 일으키는 현상.

### 7.1 수신측 SWS

```
buffer 가득 (Window=0) → 응용이 1 byte 읽음 → Window=1 광고
→ 송신자 1 byte 보냄 → 응용 1 byte 읽음 → Window=1 → 1 byte 보냄
→ 매번 40+ byte 헤더 / 1 byte 페이로드
```

### 7.2 Clark's Solution (수신측)
- Window 광고 미루기 — Window 가 MSS 또는 buffer 절반 이상 비기 전까진 Window=0 유지

### 7.3 송신측 SWS

작은 segment 안 보내기:
- **Nagle's Algorithm** — 미확인 데이터 있으면 모음 (자세히 → [[tcp]])

---

## 8. Receive Window 계산 예

```
수신 buffer 크기 (BUF): 64 KB
받은 byte 중 응용 미수신: 16 KB
Window 광고 = 64 - 16 = 48 KB

응용이 read() 호출 → 16 KB 처리
Window = 64 KB (광고)
```

### Window Update 패킷
- 데이터 없이 ACK + 큰 Window 만 전송
- 수신자가 buffer 비웠을 때

---

## 9. Flow Control vs Congestion Control

| 측면 | Flow Control | Congestion Control |
| --- | --- | --- |
| 보호 대상 | 수신자 buffer | 네트워크 |
| 신호 | Receive Window (rwnd) | Loss / ECN / RTT |
| 정의 | TCP 표준 | 알고리즘 (CUBIC/BBR) |
| 계산 | 수신자 | 송신자 |

### 최종 Window

```
W_send = min(rwnd, cwnd)
```

- `rwnd` — Receive Window (Flow Control)
- `cwnd` — Congestion Window (Congestion Control)

자세히 → [[congestion-control]]

---

## 10. 흐름 제어의 실제 시나리오

### 시나리오 1 — 슬로우 컨슈머

```
서버 → 클라 (데이터 1 GB)
클라 응용 = 디스크 IO 느림
→ 클라 buffer 가득 → Window=0
→ 서버 멈춤
→ 클라 응용 처리 → Window 광고
→ 서버 재개
```

→ TCP 가 자동으로 속도 조절 — **응용 코드는 그냥 read/write**.

### 시나리오 2 — 빠른 네트워크 + 큰 BDP

```
1 Gbps + 100ms RTT = 12.5 MB BDP

Window Scale 비활성 → 64 KB → 5.2 Mbps
Window Scale 7 → 64 KB × 128 = 8 MB → 640 Mbps
Window Scale 14 → 1 GB → 1 Gbps full
```

---

## 11. Window 의 디버깅

### 11.1 ss

```bash
ss -tin

# 출력 (관련 필드):
# ...
# cubic wscale:7,7 rto:204 rtt:7.234/5.123
# rcv_space:14600 rcv_ssthresh:64088 snd_wnd:65535 cwnd:10
# bytes_sent:1234567 bytes_received:7654321
```

- `wscale` — Window Scale (송/수신)
- `snd_wnd` — 송신 측이 본 수신자의 Window
- `cwnd` — Congestion Window

### 11.2 tcpdump

```bash
sudo tcpdump -i en0 -nn 'tcp[14:2] != 0' -v
# Window 필드 = 0 이 아닌 패킷만
```

### 11.3 Wireshark
- "TCP Window Update" 자동 표시
- "TCP Zero Window" 경고
- "TCP Window Full" 경고

---

## 12. 함정

### 함정 1 — Window Scale 협상 실패
SYN 에 Option 없으면 비활성. 미들박스가 SYN 옵션 제거 시 발생.

### 함정 2 — Buffer 너무 작음
큰 BDP 환경에서 throughput 낮음. `tcp_rmem` / `tcp_wmem` 늘림.

### 함정 3 — Buffer 너무 큼
"Bufferbloat" — 큰 buffer → ACK 지연 → 느린 혼잡 반응. AQM (CoDel) 필요.

### 함정 4 — 응용 read() 빈도 부족
응용이 천천히 read → Window 작게 광고 → throughput ↓.

### 함정 5 — Zero Window 의 영구
Window Update 손실 + Persist 안 함 → stall. 모던 Linux 는 OK.

### 함정 6 — SWS
모던 OS 는 자동 방어. 일부 임베디드 / 옛 시스템에 있음.

---

## 13. 학습 자료

- RFC 1122, RFC 9293
- RFC 1323 (Window Scale, Timestamp)
- **TCP/IP Illustrated Vol.1** Ch. 20 (Flow Control)
- **Computer Networking: Top-Down** Ch. 3.5
- Linux Kernel `tcp_input.c` `tcp_update_pacing_rate()`

---

## 14. 관련

- [[congestion-control]] — cwnd
- [[tcp-options]] — Window Scale, MSS
- [[tcp-header]] — Window 필드
- [[tcp-performance-tuning]] — sysctl 튜닝
- [[tcp]] — TCP hub
