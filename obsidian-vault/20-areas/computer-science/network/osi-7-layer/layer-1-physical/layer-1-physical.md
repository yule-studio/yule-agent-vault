---
title: "L1 — 물리 계층 (Physical Layer)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T15:10:00+09:00
tags:
  - network
  - osi
  - layer-1
  - physical-layer
---

# L1 — 물리 계층 (Physical Layer)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 깊이 분리 — Physical 개요 + 세부 노트 4 개 |

**[[../osi-7-layer|↑ OSI 7 계층]]** · **[[../../network|↑↑ network hub]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **전송 단위** | bit (0/1) |
| **주소 / 식별자** | 매체 자체 — 케이블 / 회선 |
| **대표 장치** | 케이블, 단자, 안테나, 앰프, 버스, 중계기 (Repeater), 허브 (Hub), 모뎀, NIC PHY |
| **대표 프로토콜 / 표준** | PCM, RS-232 / RS-422 / RS-485, 10BASE-T, 100BASE-TX, 1000BASE-T, 10GBASE-SR, IEEE 802.3 (PHY), IEEE 802.11 PHY, ITU-X.21 |
| **데이터 형식** | 전기적 / 광학적 / 무선 RF 신호 |
| **상위 계층** | L2 Data Link |

---

## 1. 한 줄 정의

**bit 단위의 디지털 데이터를 물리적 신호 (전기 / 광 / 무선) 로 변환** 해 매체로
전송 + 수신 측에서 다시 비트로 복원하는 계층. OSI 7 계층의 최하위.

---

## 2. 역사

| 연도 | 사건 |
| --- | --- |
| 1837 | Morse — 전신, 최초의 디지털 통신 |
| 1876 | Bell — 전화 |
| 1958 | RS-232 표준 (EIA) |
| 1973 | Robert Metcalfe — 이더넷 (Xerox PARC) |
| 1980 | IEEE 802.3 — 10BASE5 ("Thicknet") |
| 1990 | 10BASE-T (Twisted Pair Ethernet) |
| 1999 | 1000BASE-T (Gigabit) |
| 2006 | 10 Gigabit Ethernet 보편화 |
| 2017 | 400 GbE 표준 (IEEE 802.3bs) |
| 2024 | 800 GbE 데이터센터 상용화 |

---

## 3. L1 의 책임 — 5 가지

### 3.1 물리적 연결
케이블 / 단자 / 무선 안테나의 사양 정의 — 핀 배치, 전압, 임피던스.

### 3.2 신호 변환 (Encoding / Modulation)
디지털 비트 → 매체에 맞는 신호 (전류 / 광 펄스 / RF) 로 변환.
- 베이스밴드: NRZ, Manchester (유선)
- 패스밴드 (변조): ASK, FSK, PSK, QAM (무선 / 광)

자세히 → [[encoding-modulation]]

### 3.3 신호 동기화
송신 / 수신 클록 정렬 — Self-clocking (Manchester), Preamble.

### 3.4 잡음 / 손실 대응
감쇠, 지연, 잡음 — 리피터 / 증폭기 / 이퀄라이저로 보상.

자세히 → [[noise-and-errors]]

### 3.5 매체 사양
케이블 / 광섬유 / 무선 주파수 — 거리, 대역폭, 간섭 특성.

자세히 → [[transmission-media]]

---

## 4. L1 의 장치 (Devices)

### 4.1 케이블 / 안테나
순수 매체 — UTP Cat 6, OS2 single-mode fiber, dipole 안테나.

### 4.2 트랜시버 (Transceiver)
Transmitter + Receiver. NIC, SFP/SFP+ 모듈.

### 4.3 리피터 (Repeater)
감쇠된 신호를 증폭 + 재전송. 양방향. 충돌 도메인 합치는 효과.

### 4.4 허브 (Hub)
Multi-port 리피터. 하나의 충돌 도메인 → 비효율 → 스위치 (L2) 로 대체됨.

### 4.5 모뎀 (Modem)
Modulator + Demodulator. 디지털 ↔ 아날로그.

### 4.6 NIC (Network Interface Card)
PHY (L1) + MAC (L2) 가 한 칩. 호스트와 매체를 연결.

자세히 → [[cables-connectors-devices]]

---

## 5. 전송 방식

### 5.1 직렬 (Serial) vs 병렬 (Parallel)
- 직렬 — 1 비트씩 (RS-232, USB, SATA, PCIe). 장거리.
- 병렬 — 여러 비트 동시 (옛 IDE, 프린터 포트). 짧은 거리만.

현대는 거의 직렬 (skew 문제로 병렬 unsuitable at high speed).

### 5.2 단방향 / 양방향
- **Simplex** — 한 방향만 (TV 방송)
- **Half-duplex** — 양방향이지만 동시 X (Walkie-talkie, 옛 Hub)
- **Full-duplex** — 양방향 동시 (현대 Ethernet)

### 5.3 동기 / 비동기
- **Synchronous** — 클록 공유 (Ethernet PHY)
- **Asynchronous** — Start bit + Stop bit (UART, RS-232)

### 5.4 베이스밴드 (Baseband) vs 브로드밴드 (Broadband)
- Baseband — 한 채널 / 디지털 신호 그대로 (Ethernet)
- Broadband — 주파수 분할 다중 (Cable Internet, ADSL)

---

## 6. 성능 지표

### 6.1 대역폭 (Bandwidth)
주파수 폭 (Hz) — 매체가 통과시키는 주파수 범위. Shannon 한계로 비트율 결정.

### 6.2 비트율 (Bit rate)
초당 비트 — bps. 100 Mbps, 1 Gbps, 10 Gbps.

### 6.3 보율 (Baud rate)
초당 심볼 — Bit rate ≠ Baud rate (한 심볼이 여러 비트 표현 가능, QAM).

### 6.4 SNR (Signal-to-Noise Ratio)
신호 전력 / 잡음 전력 (dB). 높을수록 좋음.

### 6.5 Eb/N0
비트당 에너지 / 잡음 PSD. 디지털 변조 성능 비교 표준.

### 6.6 BER (Bit Error Rate)
오류 비트 / 전체 비트. Ethernet: 10⁻¹², Wi-Fi: 10⁻⁶~10⁻³.

### 6.7 Shannon 한계
```
C = B × log₂(1 + S/N)
```
- C: 채널 용량 (bps)
- B: 대역폭 (Hz)
- S/N: SNR
- 1948 Claude Shannon — 정보 이론 기초.

---

## 7. 매체별 전송 거리 표

| 매체 | 거리 | 속도 |
| --- | --- | --- |
| UTP Cat 5e | 100m | 1 Gbps |
| UTP Cat 6 | 100m | 1-10 Gbps |
| UTP Cat 6a / Cat 7 | 100m | 10 Gbps |
| 동축 (10BASE2) | 185m | 10 Mbps |
| 동축 (10BASE5) | 500m | 10 Mbps |
| 멀티모드 광 (OM3) | 300m | 10 Gbps |
| 싱글모드 광 (OS2) | 10-40 km | 10-400 Gbps |
| Wi-Fi 6 | 30-100m | 9.6 Gbps (이론) |
| 5G (mmWave) | <300m | 10 Gbps (이론) |
| 위성 (LEO, Starlink) | 수백 km | 100 Mbps - 1 Gbps |

---

## 8. L1 의 함정

### 함정 1 — 케이블 종류 혼동
Cat 5 vs Cat 5e vs Cat 6 — 외관 같지만 성능 차이. 데이터센터에서 1m 짧은 케이블도
스펙 확인.

### 함정 2 — 길이 제한 무시
UTP 100m 가 한계. 그 이상은 신호 열화 → 리피터 / 광 변환.

### 함정 3 — EMI / 누화
형광등 / 모터 옆 UTP → 데이터 손실. STP 또는 광섬유.

### 함정 4 — 듀얼 / 자동 협상 실패
Speed/Duplex mismatch — 한쪽 Full / 한쪽 Half → 패킷 손실, CRC 증가. 자동 협상
실패 시 양쪽 명시.

### 함정 5 — 광섬유 dirty connector
먼지 한 조각 → 광 손실 dB 단위. 청소 도구 필수.

### 함정 6 — 무선 채널 충돌
2.4 GHz 13 채널 → 실제 비중첩 1/6/11. 5 GHz / 6 GHz 권장.

---

## 9. 세부 노트 — 이 폴더의 깊이

| 노트 | 영역 |
| --- | --- |
| [[transmission-media]] | UTP / STP / 동축 / 광 / 무선 매체 깊이 |
| [[encoding-modulation]] | NRZ / Manchester / B8ZS / ASK / FSK / PSK / QAM |
| [[noise-and-errors]] | 감쇠 / 지연 / 4 종 잡음 / Shannon / BER |
| [[cables-connectors-devices]] | RJ-45 / BNC / SC / LC / SFP / NIC / 모뎀 |

---

## 10. 면접 / 토픽

1. **L1 의 책임 5 가지**.
2. **UTP vs STP vs 광섬유** — 거리, 속도, 간섭.
3. **NRZ vs Manchester** — 왜 Ethernet 에서 Manchester 였나.
4. **Shannon 한계** — 식과 의미.
5. **Half vs Full Duplex** — 충돌 도메인.
6. **무선 채널 / SNR**.

---

## 11. 학습 자료

- **Data and Computer Communications** (William Stallings)
- **High-Speed Signaling: Jitter Modeling, Analysis, and Budgeting** (Sudhakar Yalamanchili)
- **Cabling: The Complete Guide to Network Wiring** (Andrew Oliviero)
- IEEE 802.3 표준 문서 (PHY)
- Cisco — Cabling guide

---

## 12. 관련

- [[../layer-2-data-link/layer-2-data-link]] — 상위 계층
- [[../osi-7-layer|↑ OSI 7 계층]]
- [[../../hardware/hardware|↗ hardware]] — 트랜지스터 / NIC
- [[../../network|↑↑ network hub]]
