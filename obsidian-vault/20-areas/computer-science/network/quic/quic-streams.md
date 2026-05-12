---
title: "QUIC Streams & Flow Control"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T18:40:00+09:00
tags:
  - network
  - quic
  - streams
  - flow-control
---

# QUIC Streams & Flow Control

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Streams / Stream ID / 흐름 제어 / Frame |

**[[quic|↑ QUIC]]** · **[[../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

QUIC 의 **Stream** = 한 connection 안의 **독립된 양방향 byte 흐름**. 수천 개
동시 — HoL Blocking 해결의 핵심.

---

## 2. Stream ID — 식별 + 분류

```
[Stream ID 62-bit Variable Length Integer]
하위 2 bit:
  bit 0: 0 = Client initiated, 1 = Server initiated
  bit 1: 0 = Bidirectional, 1 = Unidirectional
```

| Stream ID 하위 2bit | 종류 |
| --- | --- |
| `00` | Client-initiated bidirectional |
| `01` | Server-initiated bidirectional |
| `10` | Client-initiated unidirectional |
| `11` | Server-initiated unidirectional |

### HTTP/3 에서
- Bidirectional → request/response
- Unidirectional → control / push / decoder / encoder

---

## 3. Stream State

```
Open → Receive only → Closed
Open → Send only    → Closed

각 방향 독립:
  Send: Ready → Send → Data Sent → Reset Sent → ...
  Recv: Recv → Size Known → Data Recvd → ...
```

---

## 4. Stream 생성 / 종료

### 4.1 생성
- 첫 STREAM frame 송신 시 자동 생성
- 명시적 OPEN frame X

### 4.2 종료
- **FIN bit** — 정상 종료 ("더 보낼 데이터 없음")
- **RESET_STREAM** — 비정상 종료
- **STOP_SENDING** — 받는 측이 "그만 보내"

---

## 5. STREAM Frame

```
+--------+--------+--------+
| Type   | Stream | Offset |   Type = 0x08-0x0f (bit flag)
| ID     | (var)  |        |
+--------+--------+--------+
| Length (var)             |
+--------------------------+
| Stream Data ...          |
+--------------------------+
```

### Type 의 bit flag
- `0x08 + OFF` — Offset 포함
- `0x08 + LEN` — Length 포함
- `0x08 + FIN` — 마지막

→ Stream 한 byte 단위 offset — 손실 / 재정렬 처리 쉬움.

---

## 6. HoL Blocking 해결

```
HTTP/2 over TCP:
패킷 1 (req1 data1)
패킷 2 (req2 data1) ← 손실
패킷 3 (req3 data1)
패킷 4 (req4 data1)
↓
패킷 2 손실 → TCP 가 모두 buffer → 모든 stream stuck
```

```
HTTP/3 over QUIC:
패킷 1 (stream1 + stream2 data)
패킷 2 (stream1 only) ← 손실
패킷 3 (stream2 data)
↓
stream 1 만 stuck, stream 2 는 즉시 처리
```

---

## 7. Flow Control — 2 레벨

### 7.1 Stream-level

각 stream 별 받을 수 있는 byte:
- **MAX_STREAM_DATA** frame — 수신자가 송신자에게 광고
- Stream 의 buffer 가득 시 0 알림

### 7.2 Connection-level

전체 connection 의 byte:
- **MAX_DATA** frame — 모든 stream 합산 한계

→ Stream + Connection 의 작은 쪽이 실제 한계.

### 7.3 Window Update

```
수신: 응용이 N byte 읽음 → buffer 비움
수신 → 송신: MAX_STREAM_DATA (또는 MAX_DATA) frame → 늘어난 한계
```

TCP 의 Window Update 와 같은 개념.

---

## 8. Stream 우선순위 (HTTP/3 Priority)

### 8.1 PRIORITY_UPDATE frame (HTTP/3 RFC 9218)

```
urgency: 0-7 (낮을수록 중요)
incremental: 점진 vs 한꺼번에
```

HTTP/2 의 복잡한 priority tree 가 RFC 9218 로 단순화.

### 8.2 사용
- 큰 이미지 < CSS < JS
- 사용자 입력 응답 > 백그라운드 로드

---

## 9. Stream Concurrency Limits

### 9.1 초기 한계

```
Transport Parameters (handshake 중 교환):
  initial_max_streams_bidi: 100  (예)
  initial_max_streams_uni: 3
```

### 9.2 동적 증가

```
서버가 MAX_STREAMS frame 으로 한계 늘림
```

### 9.3 함정
- 너무 작으면 응용 stuck
- 너무 크면 자원 소진 위험

---

## 10. Stream 별 RESET_STREAM

```
보내는 측: "이 stream 중단"
수신 측: 받은 데이터 폐기 OK
```

HTTP/3 의 cancel 처럼 사용.

---

## 11. Datagram (RFC 9221)

QUIC 의 비신뢰 datagram — Stream 외 채널:
- WebTransport / WebRTC 위
- 비디오 / 게임 데이터
- "QUIC 이지만 Stream 의 신뢰성 X"

---

## 12. Multiplexing 비교

| 프로토콜 | 멀티 스트림 | HoL 해결 |
| --- | --- | --- |
| HTTP/1.1 | 별개 TCP 연결 (6 개 한계) | TCP 레벨로 부분 |
| HTTP/2 | 한 TCP 연결의 stream | 응용 레벨, TCP HoL 잔존 |
| HTTP/3 | QUIC stream | 완전 해결 |

---

## 13. 함정

### 함정 1 — Stream limit 너무 작음
응용이 stream 못 만들어 stuck. 초기값 조정.

### 함정 2 — Flow control 의 작은 buffer
큰 BDP 환경에서 throughput 낮음.

### 함정 3 — Priority 무시
중요 응답이 늦게 → 사용자 경험 ↓.

### 함정 4 — Stream 의 순서 보장 가정
한 stream 안은 순서 OK, 여러 stream 간 순서 X.

### 함정 5 — Stream RESET 의 의미
응용이 cancel 의도 — 부분 처리된 결과 폐기 필요.

---

## 14. 학습 자료

- RFC 9000 Section 2-3, 19 (Stream / Frame)
- RFC 9218 (HTTP/3 Priority)
- RFC 9221 (QUIC Datagram)
- "QUIC Streams Explained" — qvis

---

## 15. 관련

- [[quic]] — QUIC hub
- [[quic-handshake]] — Handshake
- [[../http/http3-quic]] — HTTP/3 의 stream 활용
