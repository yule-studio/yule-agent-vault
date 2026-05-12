---
title: "TCP Options (MSS / SACK / Timestamp / Window Scale / TFO)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T17:55:00+09:00
tags:
  - network
  - tcp
  - tcp-options
  - sack
  - mss
---

# TCP Options (MSS / SACK / Timestamp / Window Scale / TFO)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 옵션 전체 깊이 |

**[[tcp|↑ TCP]]** · **[[../network|↑↑ network hub]]**

---

## 1. 옵션 일반

```
TCP 헤더 끝 (Data Offset > 5) 에 위치
최대 40 byte (Data Offset 15 = 60 byte 헤더 - 20 기본)

형식: [Kind (1)][Length (1)][Data]
4 byte 정렬 — NOP (Kind=1) 으로 padding
```

### IANA 등록 옵션

| Kind | 길이 | 이름 |
| --- | --- | --- |
| 0 | 1 | End of Options List (EOL) |
| 1 | 1 | NOP (No Operation) — padding |
| 2 | 4 | MSS |
| 3 | 3 | Window Scale |
| 4 | 2 | SACK Permitted |
| 5 | 가변 | SACK |
| 8 | 10 | Timestamp |
| 19 | 18 | MD5 Signature (BGP) |
| 28 | 4 | User Timeout |
| 29 | 가변 | TCP-AO (Authentication Option) |
| 34 | 가변 | TCP Fast Open Cookie |

---

## 2. MSS — Maximum Segment Size (Kind 2)

### 2.1 정의
TCP 가 한 segment 에 담을 수 있는 **payload 최대 byte**.

### 2.2 협상
- SYN 에 양쪽이 자기 MSS 광고
- 작은 쪽 채택 (이후 변경 X)
- SYN 이외엔 무시

### 2.3 계산
```
MSS = MTU - IP Header - TCP Header
    = 1500 - 20 - 20 = 1460  (표준 Ethernet)
```

VPN / PPPoE 면 더 작음.

### 2.4 PMTUD 와 상호작용
MSS 협상 후 라우터의 ICMP Type 3 Code 4 로 더 작아질 수 있음.

자세히 → [[../osi-7-layer/layer-3-network/fragmentation-mtu]]

---

## 3. Window Scale (Kind 3)

### 3.1 정의
Window 필드 (16 bit, 64 KB 한계) 를 **2^N 배 확대**.

```
실제 Window = Window 필드 × 2^ScaleFactor
ScaleFactor: 0-14 → 최대 1 GB
```

### 3.2 협상
- SYN 에 양쪽이 ScaleFactor 광고
- 양쪽 모두 보내야 활성
- 한쪽 미지원 → 둘 다 ScaleFactor = 0

### 3.3 사용
- BDP 큰 환경 (Long Fat Network)
- 모던 OS 기본 활성

자세히 → [[sliding-window#5 BDP]]

---

## 4. SACK — Selective Acknowledgment (Kind 4, 5)

### 4.1 동기

기본 누적 ACK 의 한계:
```
손실: seq 1000-1500 (segment 1) 손실
받음: seq 1500-2000 (segment 2)
받음: seq 2000-2500 (segment 3)
보낼 ACK: 1000  (1 부터 다시 보내라?)

→ Go-Back-N — segment 1, 2, 3 모두 재전송 비효율
```

### 4.2 SACK

```
ACK: 1000
SACK: [1500, 2500]  ← "1500-2500 받았어"
→ 송신자: 1000-1500 만 재전송
```

### 4.3 헤더

```
Kind=4 (SACK Permitted, SYN 시): "지원해"
Kind=5 (SACK 실제):
  [Kind=5][Length][Left Edge 1][Right Edge 1][Left 2][Right 2]...
  최대 4 블록 (Length 가변)
```

### 4.4 DSACK (Duplicate SACK, RFC 2883)
- 중복 받은 segment 알림
- 송신자가 불필요한 재전송 인지

### 4.5 협상
- SYN / SYN+ACK 에 SACK Permitted (Kind 4)
- 양쪽 모두 동의해야 SACK 사용

---

## 5. Timestamp (Kind 8)

### 5.1 정의
```
[Kind=8][Length=10][TSval (4)][TSecr (4)]
```

- **TSval** — 송신자 현재 시간 (랜덤 offset OK)
- **TSecr** — 마지막 받은 TSval echo

### 5.2 용도 1 — RTT 정확 측정

```
A → B: data + TSval=100
B → A: ACK + TSval=200, TSecr=100
A 가 ACK 받을 때 시간 200 → RTT = 200 - 100 = 100 ms
```

재전송된 segment 의 ACK 가 와도 정확 측정 (Karn's algorithm 문제 해결).

### 5.3 용도 2 — PAWS (Protect Against Wrapped Sequence)

Seq# 가 wrap 했을 때 옛 패킷 / 새 패킷 구분:
- 옛 패킷의 TSval 은 작음 → 폐기
- 새 패킷의 TSval 은 큼 → 수용

빠른 링크 (40G+) 에서 Seq# 가 한 RTT 안에 wrap → 필수.

### 5.4 협상
- SYN / SYN+ACK 에 Timestamp 옵션
- 양쪽 모두 보내야 활성

---

## 6. TCP Fast Open (TFO, Kind 34)

### 6.1 동기
3-way handshake = 1 RTT 낭비. 짧은 HTTP 요청에 치명적.

### 6.2 흐름

**첫 연결 (Cookie 획득)**:
```
C → S: SYN + TFO Cookie Request (empty)
S → C: SYN+ACK + TFO Cookie (서버 생성)
C → S: ACK
```

**두 번째 이후 (0-RTT 데이터)**:
```
C → S: SYN + TFO Cookie + DATA  ← 즉시 데이터
S → C: SYN+ACK + DATA            ← 즉시 응답
C → S: ACK + DATA
```

### 6.3 Cookie

```
[Kind=34][Length][Cookie 4-16 byte]
서버가 HMAC(클라 IP, secret) 로 생성
유효 시간: 보통 1 시간
```

### 6.4 사용
- HTTPS / API
- **POST 등 비멱등 요청 금지** — 재전송 위험
- Linux 3.7+ 지원

### 6.5 QUIC 의 0-RTT 가 표준
QUIC 는 표준으로 0-RTT — TFO 는 거의 사장.

---

## 7. MD5 Signature (Kind 19, RFC 2385)

### 7.1 동기
- BGP 의 인증
- TCP 헤더 변조 방지

### 7.2 동작
- 각 segment 에 MD5(secret + headers + data)
- BGP peer 끼리 사전 공유 키

### 7.3 단점
- MD5 약함
- 키 교체 어려움
- → **TCP-AO** (Kind 29, RFC 5925) 가 대체

---

## 8. User Timeout (Kind 28, RFC 5482)

응용이 "이 연결 N 초 응답 없으면 timeout" 명시.

기본 `tcp_user_timeout`:
- Linux: 0 (재전송 RTO 와 함께 알아서)
- 명시적 설정으로 빠른 실패 가능

---

## 9. TCP-AO (Kind 29, RFC 5925)

MD5 Signature 의 모던 후속:
- AES-CMAC, HMAC-SHA-256
- 키 교체 (Key ID)
- BGP, LDP 등 사용

---

## 10. 옵션 예 (SYN 패킷)

```
Wireshark 캡처:
Options: (32 byte)
  - Maximum segment size: 1460 bytes  (Kind 2, Length 4)
  - SACK permitted                    (Kind 4, Length 2)
  - Timestamp: TSval 12345 TSecr 0    (Kind 8, Length 10)
  - NOP                                (Kind 1, padding)
  - Window scale: 7 (multiply by 128) (Kind 3, Length 3)
  - End of Options List               (Kind 0)
```

---

## 11. 함정

### 함정 1 — Options 의 정렬
4 byte 정렬 — NOP padding 필수. 코드 작성 시 헷갈림.

### 함정 2 — 미들박스가 옵션 제거
일부 NAT / 방화벽이 모르는 옵션 stripping → 양쪽 미지원처럼 작동.
주범: Window Scale, SACK, Timestamp, TFO.

### 함정 3 — Timestamp 무력화로 PAWS 실패
40G+ 빠른 링크에서 Seq# wrap 시 corruption. Timestamp 필수.

### 함정 4 — TFO 의 비멱등 요청
재전송 시 두 번 처리. GET 만 안전, POST 금지.

### 함정 5 — MD5 Signature 사용
MD5 약함. TCP-AO 로 마이그레이션.

### 함정 6 — Window Scale 의 초기 광고
큰 ScaleFactor 광고 + 작은 buffer → 송신자 혼란.

### 함정 7 — Options 의 SYN 외 패킷
SYN 이후 새 옵션 무의미 (대부분 무시). 협상은 SYN 에서만.

---

## 12. 학습 자료

- RFC 793, RFC 7414 (Options Roadmap)
- RFC 2018 (SACK), RFC 1323 (Window Scale + Timestamp)
- RFC 7413 (TCP Fast Open)
- **TCP/IP Illustrated Vol.1** Ch. 17.4
- IANA TCP Options 등록

---

## 13. 관련

- [[tcp-header]] — Options 위치
- [[sliding-window]] — Window Scale
- [[congestion-control]] — SACK 활용
- [[tcp]] — TCP hub
