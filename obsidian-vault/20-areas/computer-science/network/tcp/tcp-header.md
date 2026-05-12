---
title: "TCP 헤더 (Header Structure)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T17:25:00+09:00
tags:
  - network
  - tcp
  - layer-4
  - tcp-header
---

# TCP 헤더 (Header Structure)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 헤더 필드 / Flags / Checksum |

**[[tcp|↑ TCP]]** · **[[../network|↑↑ network hub]]**

---

## 1. 헤더 전체

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Data |       |C|E|U|A|P|R|S|F|                               |
| Offset| Reserv|W|C|R|C|S|S|Y|I|            Window             |
|       |       |R|E|G|K|H|T|N|N|                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options                    |    Padding    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             data                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

기본 20 byte + Options (최대 40 byte) = 60 byte 최대.

---

## 2. 필드별 상세

### 2.1 Source Port (16 bit)

- 송신자의 응용 포트
- 클라이언트면 보통 ephemeral (49152-65535)
- 서버면 well-known (80, 443)

### 2.2 Destination Port (16 bit)

- 수신자의 응용 포트
- 응용 식별

### 2.3 Sequence Number (32 bit) — Seq

- **이 segment 의 첫 byte 의 번호**
- SYN 패킷에선 ISN (Initial Sequence Number, 랜덤)
- 매 byte 마다 +1
- Wrap-around 가능 (PAWS 가 보호)

#### 보안 — Random ISN
- 초기 RFC 793 의 timer 기반 ISN → 추측 가능 → 1985 Morris 공격
- RFC 6528 — 강한 랜덤 ISN

### 2.4 Acknowledgment Number (32 bit) — ACK

- **다음 기대 byte 의 Seq#** (누적 ACK)
- ACK flag 설정 시만 유효
- 받지 못한 첫 byte 의 번호

예: Seq 100 부터 50 byte 받음 → ACK = 150 ("150 부터 보내")

### 2.5 Data Offset (4 bit)

- TCP 헤더 길이 / 4 (32-bit word 수)
- 5-15 (= 20-60 byte)
- Options 가 있으면 5 초과

### 2.6 Reserved (4 bit 또는 6 bit)

- 0 으로 설정
- 3 bit 은 ECN 으로 재사용 (CWR / ECE + NS)

### 2.7 Flags (9 bit)

| Flag | 의미 |
| --- | --- |
| **NS** | Nonce Sum (ECN, RFC 3540) |
| **CWR** (Congestion Window Reduced) | 송신자가 cwnd 감소 알림 |
| **ECE** (ECN Echo) | 라우터의 혼잡 알림 |
| **URG** (Urgent) | Urgent Pointer 유효 |
| **ACK** | ACK Number 유효 |
| **PSH** (Push) | 버퍼링 X, 즉시 응용에 |
| **RST** (Reset) | 연결 끊기 (이상 상황) |
| **SYN** | 연결 수립 (sequence number 동기) |
| **FIN** | 연결 종료 |

#### 흔한 조합

- `SYN` — 첫 handshake
- `SYN+ACK` — 두 번째 handshake
- `ACK` — 일반 ACK
- `PSH+ACK` — 데이터 + ACK
- `FIN+ACK` — 종료 시작
- `RST+ACK` — 비정상 종료

### 2.8 Window (16 bit) — Receive Window

- "내가 받을 수 있는 byte 수"
- 흐름 제어의 핵심
- 0-65535 byte (Window Scale 옵션으로 확장 가능)
- 0 이면 송신자 멈춤 (Zero Window)

자세히 → [[sliding-window]]

### 2.9 Checksum (16 bit)

- TCP 헤더 + 데이터 + Pseudo Header (IP src/dst, protocol) 의 1's complement
- 약함 (16 bit) — Ethernet CRC + IP checksum 이 주

### 2.10 Urgent Pointer (16 bit)

- URG flag 설정 시 유효
- Sequence Number + Urgent Pointer = 긴급 데이터 끝
- 거의 사용 X (구현마다 해석 다름)

### 2.11 Options (가변, 최대 40 byte)

자세히 → [[tcp-options]]

| Kind | 길이 | 옵션 |
| --- | --- | --- |
| 0 | — | End of Options List |
| 1 | — | NOP (Padding) |
| 2 | 4 | MSS (Maximum Segment Size) |
| 3 | 3 | Window Scale |
| 4 | 2 | SACK Permitted |
| 5 | 가변 | SACK |
| 8 | 10 | Timestamp |
| 19 | 18 | MD5 Signature |
| 28 | 4 | User Timeout |
| 29 | 가변 | TCP-AO (Authentication) |
| 34 | 가변 | TCP Fast Open Cookie |

### 2.12 Data

실제 페이로드 — HTTP, SSH 등.

---

## 3. Pseudo Header (Checksum 용)

```
+--------+--------+--------+--------+
|           Source IP               |
+--------+--------+--------+--------+
|         Destination IP            |
+--------+--------+--------+--------+
|  zero  | Proto  |   TCP Length    |
+--------+--------+--------+--------+
```

L3 (IP) 의 일부 정보가 L4 checksum 에 포함 → IP 가 변조되면 checksum 깨짐.

NAT 는 IP / Port 변경 시 checksum 재계산 필수.

---

## 4. 헤더의 의미 — 패킷 사례

### 4.1 SYN 패킷
```
Src Port: 49152
Dst Port: 80
Seq#: 1000 (ISN, 랜덤)
ACK#: 0
Flags: SYN
Window: 65535
Options: MSS=1460, SACK_PERM, Timestamp, Window Scale=7
```

### 4.2 SYN+ACK 패킷
```
Src Port: 80
Dst Port: 49152
Seq#: 2000 (서버 ISN)
ACK#: 1001  (= 클라 Seq + 1, SYN 도 1 byte 로 침)
Flags: SYN, ACK
Window: 28960
Options: MSS=1460, SACK_PERM, Timestamp echo, Window Scale=14
```

### 4.3 ACK 패킷
```
Src Port: 49152
Dst Port: 80
Seq#: 1001
ACK#: 2001
Flags: ACK
Window: 65535
```

### 4.4 데이터 패킷 (HTTP GET)
```
Src Port: 49152
Dst Port: 80
Seq#: 1001
ACK#: 2001
Flags: PSH, ACK
Window: 65535
Data: "GET / HTTP/1.1\r\nHost: ..."
```

### 4.5 RST 패킷 (포트 closed)
```
Src Port: 80
Dst Port: 49152
Flags: RST
Seq#: 0
ACK#: 1001
```

---

## 5. 헤더 크기 계산 — 함정

```
Ethernet (14) + IP (20) + TCP (20) = 54 byte 헤더
Ethernet (14) + IP (20) + TCP (32 with Options) = 66 byte

Path MTU 1500 → TCP payload 최대:
  1500 - 20 - 20 = 1460 byte (MSS)
  Options 12 byte 있으면 → 1460 - 12 = 1448 byte
```

---

## 6. Wireshark 로 헤더 보기

```
display filter:
  tcp                                  (모든 TCP)
  tcp.flags.syn == 1 and tcp.flags.ack == 0     (SYN only)
  tcp.flags.fin == 1                   (FIN)
  tcp.flags.reset == 1                 (RST)
  tcp.port == 443                      (HTTPS)
  tcp.stream eq 5                      (특정 연결)
```

---

## 7. 함정

### 함정 1 — Seq# wrap-around
4GB+ 빠른 링크에서 Seq# 가 한 RTT 안에 wrap → 옛 패킷이 새 것으로 오인. PAWS (Timestamp) 가 보호.

### 함정 2 — ISN 추측
약한 ISN → Sequence prediction attack. 모던 OS 는 강한 랜덤.

### 함정 3 — Window 의 단위 (Window Scale 적용)
실제 Window = Window 필드 × 2^ScaleFactor.

### 함정 4 — Checksum 의 약함
16-bit Internet Checksum — 일부 byte swap 못 잡음. IP / Ethernet 의 CRC 가 보완.

### 함정 5 — Options 의 정렬
4 byte 정렬 필요 — NOP 로 padding.

### 함정 6 — URG / Urgent Pointer
구현마다 다름. 거의 안 씀.

---

## 8. 학습 자료

- RFC 793 / RFC 9293
- **TCP/IP Illustrated Vol.1** Ch. 17
- Wireshark "TCP Analysis" 기능
- RFC 6093 (Urgent Mechanism), RFC 7414 (TCP Options Roadmap)

---

## 9. 관련

- [[tcp]] — TCP hub
- [[three-way-handshake]] — SYN/SYN-ACK/ACK
- [[tcp-options]] — Options 깊이
- [[sliding-window]] — Window 깊이
