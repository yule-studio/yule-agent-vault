---
title: "DTE / DCE / 인터페이스 — RS-232, RS-422, RS-485"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T16:15:00+09:00
tags:
  - network
  - layer-1-2
  - rs-232
  - rs-485
  - serial
---

# DTE / DCE / 인터페이스 — RS-232, RS-422, RS-485

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | DTE/DCE / RS-232/422/485 / X.21 / V.35 / DSU |

**[[layer-2-data-link|↑ L2 Data Link]]** · **[[../osi-7-layer|↑↑ OSI 7]]**

> 시리얼 인터페이스 표준 — 옛 모뎀, 산업 / 임베디드, WAN 의 기본.
> L1 과 L2 사이 — 핀 / 전압 (L1) + 흐름 제어 (L2).

---

## 1. DTE vs DCE

### 1.1 DTE (Data Terminal Equipment)

- **데이터를 만드는 / 받는 장치**
- 컴퓨터, 단말기, Router 등
- Pin 정의에서 신호 송신 방향이 다름

### 1.2 DCE (Data Circuit-terminating Equipment / Data Communications Equipment)

- **DTE 와 매체 사이의 변환기**
- 모뎀, CSU/DSU, 라우터의 일부 포트
- DTE 의 데이터를 매체에 맞게 변환

### 1.3 구조

```
DTE ─ Cable ─ DCE ─── 매체 (전화선/X.25 등) ─── DCE ─ Cable ─ DTE
컴퓨터        모뎀                                  모뎀         컴퓨터
```

### 1.4 동기 vs 비동기

- **비동기 (Async)** — 클록 없음, Start/Stop bit. RS-232 의 흔한 모드.
- **동기 (Sync)** — 클록 공유. RS-422, X.21, V.35.

DCE 가 동기 모드에서 **클록을 제공** — DTE 가 그 클록에 맞춤.

---

## 2. RS-232 (EIA-232, 1958)

### 2.1 역사 / 개요

- 1958 EIA (Electronic Industries Alliance) 표준
- 첫 시리얼 통신 표준
- 컴퓨터 ↔ 모뎀 연결
- 한 가닥 케이블당 1:1 (Point-to-Point)

### 2.2 전기 특성

| 신호 | 전압 |
| --- | --- |
| `1` (Mark) | -3V ~ -25V |
| `0` (Space) | +3V ~ +25V |
| 무신호 | -3V ~ +3V (금지 영역) |

NRZ-L, 단일 종단 (Unbalanced).

### 2.3 핀 배치 (DB-25 / DB-9)

#### DB-9 (가장 흔함)

| 핀 | 신호 | 방향 (DTE 기준) | 의미 |
| --- | --- | --- | --- |
| 1 | CD / DCD | ← | Carrier Detect |
| 2 | RXD / RD | ← | Receive Data |
| 3 | TXD / TD | → | Transmit Data |
| 4 | DTR | → | Data Terminal Ready |
| 5 | GND | — | Signal Ground |
| 6 | DSR | ← | Data Set Ready |
| 7 | RTS | → | Request To Send |
| 8 | CTS | ← | Clear To Send |
| 9 | RI | ← | Ring Indicator |

#### DB-25 (옛)
같은 신호 + 추가 (2차 채널 등). 25 핀 중 9 만 보통 사용.

### 2.4 흐름 제어

| 종류 | 메커니즘 |
| --- | --- |
| **None** | 흐름 제어 없음 |
| **Hardware (RTS/CTS)** | RTS = "보낼게", CTS = "OK" |
| **Software (XON/XOFF)** | XON = 0x11, XOFF = 0x13 |

### 2.5 통신 파라미터

`9600-8N1`:
- 9600 — baud rate
- 8 — data bits (7 or 8)
- N — parity (N/E/O — None/Even/Odd)
- 1 — stop bits (1, 1.5, 2)

### 2.6 한계

- 거리 **15m** (50 ft) 표준
- 속도 최대 약 115.2 kbps (실용)
- 1:1 만

### 2.7 사용

- 옛 모뎀 (dial-up)
- 임베디드 콘솔 (네트워크 장비, 라우터, 가전)
- GPS 모듈
- POS / 산업 장비
- USB-Serial 변환기 (FTDI, CH340)

### 2.8 Null Modem Cable

DTE ↔ DTE 직접 연결 시 TX/RX swap 한 케이블:
- DTE A's TX → DTE B's RX
- DTE A's RX ← DTE B's TX

옛 노트북 ↔ 노트북 파일 전송에 사용.

---

## 3. RS-422 (TIA/EIA-422)

### 3.1 차이점 (vs RS-232)

| 기준 | RS-232 | RS-422 |
| --- | --- | --- |
| 신호 | 단일 종단 (Unbalanced) | **차동 (Differential)** |
| 거리 | 15m | **1.2 km** |
| 속도 | 115 kbps | **10 Mbps** |
| 잡음 | 약함 | 강함 (차동) |
| 노드 | 1:1 | 1:10 (1 송신, 10 수신) |

### 3.2 차동 신호

```
A 와 B 두 선의 전압 차이로 비트:
  V(A) - V(B) > +0.2V → 1
  V(A) - V(B) < -0.2V → 0

외부 EMI 는 양 선에 같이 영향 → 차이는 그대로 → 강함
```

### 3.3 사용
- 산업 자동화
- 옛 Apple LocalTalk
- DMX512 (조명 제어)

---

## 4. RS-485 (TIA/EIA-485)

### 4.1 차이점 (vs RS-422)

| 기준 | RS-422 | RS-485 |
| --- | --- | --- |
| 노드 | 1 송신 + 10 수신 | **32 송신 + 32 수신** (Multi-drop) |
| 양방향 | 별도 페어 | **2-wire half-duplex** (1 페어) |
| 거리 | 1.2 km | 1.2 km |
| 속도 | 10 Mbps | 10 Mbps |

### 4.2 Multi-drop

```
─── A ─── A ─── A ─── A ─── ...
─── B ─── B ─── B ─── B ─── ...
   Node1  Node2  Node3  Node4

한 페어에 32 노드 — 차례로 송신
```

### 4.3 사용

- **Modbus RTU** — 산업 자동화 표준
- **Profibus** — 독일 산업
- **DALI** — 조명 제어
- **DMX512** (RS-485 변형)
- **BACnet MS/TP** — 빌딩 자동화

### 4.4 종단 저항 (Termination)

- 양 끝에 120Ω 저항
- 신호 반사 방지

---

## 5. X.21, V.35 (WAN 표준)

### 5.1 X.21
- ITU-T 동기 시리얼
- 15 핀, 차동 신호
- 옛 X.25 패킷 스위칭

### 5.2 V.35
- 옛 WAN 표준 — 56 kbps - 2 Mbps
- 34 핀 Winchester 커넥터
- T1 / E1 연결

### 5.3 EIA-530 / EIA-449
- 더 빠른 시리얼 (2 Mbps+)
- DB-37 / DB-9 콤보

---

## 6. CSU/DSU (Channel Service Unit / Data Service Unit)

### 6.1 DSU
- 사용자 디지털 신호 ↔ T1/E1 라인
- 동기화, framing

### 6.2 CSU
- 통신사 회선과의 인터페이스
- Loop back 테스트
- 신호 보호

### 6.3 통합
모던 라우터는 CSU/DSU 내장 (Cisco WIC).

---

## 7. UART (Universal Async Receiver-Transmitter)

### 7.1 UART vs RS-232

| 측면 | UART | RS-232 |
| --- | --- | --- |
| 정의 | 칩 / 프로토콜 | 전기 표준 |
| 전압 | TTL (0/3.3V or 5V) | ±12V |
| 거리 | 짧음 (PCB 안) | 15m |

UART → 레벨 시프터 (MAX232) → RS-232.

### 7.2 임베디드에서 흔함
- Arduino TX/RX
- Raspberry Pi GPIO 14/15
- ESP32 UART
- 디버그 콘솔 (115200 baud 흔함)

---

## 8. SPI / I²C (단거리 시리얼)

| 프로토콜 | 핀 | 속도 | 거리 |
| --- | --- | --- | --- |
| **SPI** | 4 (MOSI/MISO/SCK/CS) | 수 MHz | PCB 안 |
| **I²C** | 2 (SDA/SCL) | 100 kHz - 5 MHz | PCB 안 |
| **UART** | 2-3 | 수 Mbps | PCB ~ 15m |
| **RS-485** | 2 | 10 Mbps | 1.2 km |
| **USB** | 4 | Gbps | 5m (USB 3.x) |
| **CAN** | 2 | 1-5 Mbps | 수십 m (자동차) |

L2 라기보단 L1-L2 통합 임베디드 프로토콜.

---

## 9. 실용 — 라우터 콘솔 접속

```bash
# Cisco 라우터 콘솔
# 9600-8N1, No flow control

# macOS / Linux
sudo screen /dev/tty.usbserial-XXXX 9600
sudo minicom -D /dev/ttyUSB0 -b 9600

# 종료
Ctrl-A K (screen)
Ctrl-A Q (minicom)
```

USB-to-Serial 어댑터 (FTDI, Prolific, CH340) + Cisco "rollover" 케이블 (RJ-45 ↔ DB-9).

---

## 10. 함정

### 함정 1 — DTE / DCE 혼동
같은 종류끼리 직접 연결 시 null modem 필요. 모뎀(DCE) 끼리도 마찬가지.

### 함정 2 — Baud rate / Bit rate 혼동
RS-232 는 1 심볼 = 1 비트라 같지만 일반적으로 ≠.

### 함정 3 — 양 끝 파라미터 불일치
9600-8N1 vs 9600-7E2 → 깨진 글자만. 같은 설정 필수.

### 함정 4 — 흐름 제어 mismatch
한쪽 RTS/CTS, 한쪽 None → buffer overflow.

### 함정 5 — 전압 직접 연결
TTL UART (3.3V) ↔ RS-232 (±12V) 직결 → MCU 손상. MAX232 필수.

### 함정 6 — 종단 저항 누락
RS-485 의 종단 저항 없으면 reflection → 데이터 깨짐.

### 함정 7 — Modbus RTU 의 silent gap
3.5 character time 동안 silence — 잘못된 timing 으로 frame 깨짐.

---

## 11. 학습 자료

- TIA/EIA-232-F, TIA/EIA-422-B, TIA/EIA-485 표준
- **Serial Port Complete** (Jan Axelson)
- B&B Electronics RS-485 백서
- Modbus / Profibus 사양서

---

## 12. 관련

- [[../layer-1-physical/cables-connectors-devices]] — 커넥터 / 모뎀
- [[flow-control]] — RTS/CTS / XON-XOFF
- [[layer-2-data-link]] — 상위
