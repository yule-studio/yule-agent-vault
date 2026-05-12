---
title: "TCP — Transmission Control Protocol (Hub)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T17:20:00+09:00
tags:
  - network
  - tcp
  - layer-4
---

# TCP — Transmission Control Protocol (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | TCP hub + 10 개 세부 노트 |

**[[../network|↑ network hub]]** · **[[../osi-7-layer/layer-4-transport/layer-4-transport|↑↑ L4 Transport]]**

---

## 1. 한 줄 정의

**연결 지향 + 신뢰성** 보장의 종단-종단 (end-to-end) 전송 프로토콜.
RFC 793 (1981) — 인터넷의 양대 기둥 중 하나. 모든 신뢰성 응용 (HTTP, SSH, FTP) 의 토대.

---

## 2. 역사

| 연도 | 사건 |
| --- | --- |
| 1973 | Vinton Cerf / Bob Kahn — TCP 설계 |
| 1974 | RFC 675 — 첫 TCP 명세 |
| 1981 | **RFC 793** — TCP 표준 (Postel) |
| 1988 | Van Jacobson — Congestion Control (Tahoe) |
| 1990 | Reno (Fast Retransmit + Fast Recovery) |
| 1996 | RFC 2001 — Slow Start, Congestion Avoidance |
| 1998 | NewReno (RFC 2582) |
| 2000 | SACK (RFC 2018) |
| 2003 | TCP Vegas, Westwood |
| 2008 | **CUBIC** — Linux 기본 |
| 2014 | TCP Fast Open (RFC 7413) |
| 2016 | **BBR** (Google) |
| 2022 | RFC 9293 — 통합 TCP 표준 |

---

## 3. TCP 가 제공하는 것 (요약)

1. **연결 지향** — 3-way handshake 로 수립, 4-way 로 종료
2. **신뢰성** — Seq# + ACK# + 재전송
3. **순서 보장** — Seq# 로 재배열
4. **흐름 제어** — Receive Window (수신자 보호)
5. **혼잡 제어** — Congestion Window (네트워크 보호)
6. **Multiplexing** — Port 로 응용 식별
7. **Full-duplex** — 양방향 동시
8. **오류 검출** — Checksum (16 bit)

---

## 4. TCP 의 4 가지 기둥

### 4.1 헤더 (Header)
20-60 byte. Sequence Number, ACK Number, Flags, Window Size, Checksum, Options.

자세히 → [[tcp-header]]

### 4.2 연결 수립 / 종료

```
3-way handshake: SYN → SYN-ACK → ACK
4-way termination: FIN → ACK → FIN → ACK + TIME_WAIT
```

자세히 → [[three-way-handshake]], [[four-way-termination]]

### 4.3 상태 머신 (11 상태)
LISTEN, SYN_SENT, SYN_RECEIVED, ESTABLISHED, FIN_WAIT_1, FIN_WAIT_2, CLOSE_WAIT, CLOSING, LAST_ACK, TIME_WAIT, CLOSED.

자세히 → [[tcp-state-machine]]

### 4.4 흐름 / 혼잡 제어

- **Flow Control** — Receive Window (수신자 버퍼)
- **Congestion Control** — Slow Start / AIMD / CUBIC / BBR

자세히 → [[sliding-window]], [[congestion-control]]

---

## 5. TCP 의 신뢰성 메커니즘

### 5.1 Sequence Number

- 첫 byte 의 번호 (랜덤 초기값 ISN)
- 매 byte 마다 +1
- 32 bit (4 GB 까지)
- Wrap-around 가능 (PAWS — Protect Against Wrapped Sequence)

### 5.2 Acknowledgment

- 다음 기대 Seq#
- 누적 ACK — "이 이전 모두 받음"
- SACK 옵션으로 선택적 (RFC 2018)

### 5.3 재전송

- **타이머 만료 (RTO)** — Retransmission Timeout
- **빠른 재전송 (Fast Retransmit)** — 3 중복 ACK 시 즉시
- **RTO 계산** — Jacobson 알고리즘

### 5.4 RTO 계산 (Karn / Jacobson)

```
RTT 측정 → SRTT (Smoothed) + RTTVAR (variance)

α = 0.125, β = 0.25
SRTT = (1-α) × SRTT + α × RTT_sample
RTTVAR = (1-β) × RTTVAR + β × |SRTT - RTT_sample|

RTO = SRTT + 4 × RTTVAR
RTO ∈ [1초, 60초]
```

### 5.5 PSH (Push) Flag
- "데이터 즉시 응용에 전달"
- Nagle algorithm 우회

### 5.6 URG (Urgent) Flag + Urgent Pointer
- 긴급 데이터 (out-of-band)
- 거의 사용 X (구현마다 다름)

---

## 6. Nagle Algorithm (RFC 896, 1984)

작은 패킷 폭증 방지:

```
규칙: 미확인 데이터가 있고 보낼 데이터가 MSS 미만이면 → 대기 (ACK 또는 buffer 채울 때까지)
```

### 6.1 효과
- "Tinygram problem" 해결 (Telnet 1 byte 보낼 때마다 40 byte 헤더)

### 6.2 단점
- Latency 증가 — 실시간 응용에 안 좋음
- `TCP_NODELAY` 옵션으로 비활성

### 6.3 함정 — Nagle + Delayed ACK
- 송신자: Nagle 로 작은 packet 모음 → 대기
- 수신자: Delayed ACK 로 ACK 200 ms 지연
- → 200 ms latency

→ 게임 / VoIP / RPC 는 `TCP_NODELAY` 권장.

---

## 7. Delayed ACK (RFC 1122)

```
수신자가 ACK 를 즉시 X — 최대 200-500 ms 대기
→ 다른 ACK 와 합치거나 응답 데이터에 piggyback
```

### 단점
- 위 Nagle + Delayed ACK 함정

---

## 8. TCP Options

자세히 → [[tcp-options]]

| Option | 용도 |
| --- | --- |
| **MSS** | 최대 segment 크기 |
| **Window Scale** | Window > 64KB (Long Fat Networks) |
| **SACK** | 선택적 ACK |
| **Timestamp** | RTT 정확 측정 + PAWS |
| **TCP Fast Open (TFO)** | SYN 에 데이터 |
| **MD5 Signature** | BGP 인증 |

---

## 9. TCP 사용 시나리오

| 시나리오 | 적합 |
| --- | --- |
| HTTP / HTTPS | ✅ 표준 |
| SSH | ✅ 표준 |
| FTP / SMTP / IMAP | ✅ |
| 파일 전송 | ✅ |
| 데이터베이스 연결 | ✅ |
| DNS (큰 응답) | ✅ |
| 게임 (실시간) | ❌ (HoL Blocking) — UDP / QUIC |
| VoIP / 화상 | ❌ — UDP / RTP |
| IoT / 센서 | ⚠️ — MQTT 는 TCP, CoAP 는 UDP |

---

## 10. 세부 노트 — 이 폴더의 깊이

| 노트 | 영역 |
| --- | --- |
| [[tcp-header]] | 헤더 필드 / Flags / Checksum |
| [[three-way-handshake]] | 연결 수립 / SYN flood / SYN cookie |
| [[four-way-termination]] | 종료 / TIME_WAIT / Half-close |
| [[tcp-state-machine]] | 11 상태 + 전이 다이어그램 |
| [[sliding-window]] | 슬라이딩 윈도우 + Flow Control |
| [[congestion-control]] | Slow Start / AIMD / CUBIC / BBR / Reno / NewReno |
| [[tcp-options]] | MSS / SACK / TFO / Window Scale |
| [[tcp-keepalive]] | Keep-alive / Reset / Half-open |
| [[tcp-performance-tuning]] | sysctl / buffer / BDP / 튜닝 |
| [[tcp-attacks]] | SYN flood / RST attack / Sequence prediction |

---

## 11. 면접 핵심 질문

1. **TCP 가 UDP 와 다른 점**.
2. **3-way handshake 의 의미** — 왜 3 번?
3. **4-way termination + TIME_WAIT** — 왜?
4. **Seq # / ACK # 의 역할**.
5. **혼잡 제어** — Slow Start / Congestion Avoidance / Fast Retransmit / Fast Recovery.
6. **CUBIC vs BBR**.
7. **Nagle + Delayed ACK 함정**.
8. **SACK 의 의미**.
9. **TCP HoL Blocking** 과 QUIC 의 해결.
10. **TIME_WAIT 의 동작과 SO_REUSEADDR**.

---

## 12. 학습 자료

- RFC 793 (TCP), RFC 9293 (TCP 통합)
- RFC 5681 (Congestion Control)
- **TCP/IP Illustrated Vol.1** Ch. 17-24 — Stevens (필수)
- **TCP/IP Illustrated Vol.2** Ch. 14+ — 구현
- **Effective TCP/IP Programming** (Snader)
- Linux Kernel tcp_input.c, tcp_output.c — 소스

---

## 13. 관련

- [[../osi-7-layer/layer-4-transport/layer-4-transport]] — L4 hub
- [[../udp/udp]] — UDP 비교
- [[../quic/quic]] — QUIC (TCP 의 모던 대안)
- [[../http/http]] — HTTP 가 TCP 위
