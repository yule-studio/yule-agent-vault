---
title: "인코딩 & 변조 (Encoding & Modulation)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T15:20:00+09:00
tags:
  - network
  - layer-1
  - encoding
  - modulation
  - manchester
  - qam
---

# 인코딩 & 변조 (Encoding & Modulation)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 디지털 인코딩 + 디지털→아날로그 변조 + 아날로그→디지털 (PCM) |

**[[layer-1-physical|↑ L1 Physical]]** · **[[../osi-7-layer|↑↑ OSI 7]]**

---

## 0. 인코딩과 변조의 차이

| 용어 | 의미 |
| --- | --- |
| **Encoding (인코딩)** | 디지털 비트 → 디지털 신호 — 베이스밴드. (NRZ, Manchester, B8ZS) |
| **Modulation (변조)** | 디지털 / 아날로그 → 아날로그 신호 (캐리어에 실음). (ASK, FSK, PSK, QAM) |
| **Sampling (샘플링)** | 아날로그 → 디지털 (PCM) |

---

## 1. 디지털 → 디지털 인코딩 (Line Coding)

매체 위에 디지털 비트를 어떻게 전압 / 광 펄스로 표현할 것인가.

### 1.1 NRZ (Non-Return-to-Zero) 계열

#### 1.1.1 NRZ-L (Level)
```
0  →  Low  (0V or -V)
1  →  High (+V)

bit:    1 0 1 1 0 0 1 0
NRZ-L:  ▔_▔▔__▔_
```

- 가장 단순 — 2 전압 레벨
- 단점: **DC 성분** 큼 (모두 1 이면 +V 지속) → 트랜스포머 통과 X
- 단점: 동기 손실 — 긴 연속 0 / 1 에서 클록 추출 불가
- 사용: 컴퓨터 내부 단거리 (RS-232 의 일부)

#### 1.1.2 NRZ-I (Inverted)
```
0  →  변화 없음 (그대로)
1  →  전이 (transition) — 반전

bit:    1 0 1 1 0 0 1 0
NRZ-I:  ▔__▔▔▔___▔  (전이로 1 표현)
```

- 차동 (differential) — 절대 전압 X, 변화로 의미
- 케이블 극성 반전에 강함
- 여전히 긴 0 에서 동기 손실
- 사용: USB, Fast Ethernet (100BASE-FX 일부)

### 1.2 Manchester Encoding

각 비트 슬롯의 **중간** 에서 반드시 전이.

```
0  →  High → Low  (하강)
1  →  Low → High  (상승)

bit:        1     0     1     1     0     0     1
Manchester: _▔   ▔_   _▔   _▔   ▔_   ▔_   _▔
                              ↑ 중간에 전이
```

- **Self-clocking** — 매 비트 전이 → 클록 추출 가능
- DC 성분 없음 → 트랜스포머 OK
- 단점: **2 × 대역폭** 필요 (전이가 잦음)
- 사용: **10BASE-T Ethernet** (10 Mbps), 일부 무선

### 1.3 Differential Manchester

NRZ-I 처럼 변화로 의미.

```
0  →  비트 시작에 전이 + 중간 전이
1  →  비트 중간 전이만

bit:        0     1     0     1
DiffMan:   ▔_   __▔▔  ▔_   __▔▔
            ↑전이 ↑전이없음  ↑전이
```

- Self-clocking + 차동 (극성 무관)
- 사용: **Token Ring** (IEEE 802.5)

### 1.4 4B/5B & 8B/10B

긴 0 / 1 연속을 깨기 위한 **block code**.

#### 4B/5B (100BASE-TX, FDDI)
- 4 비트 → 5 비트로 매핑
- 최대 3 연속 0 보장
- 16 코드 (data) + 8 control (Idle, J, K, ...)
- 25% 오버헤드

| 4 비트 | 5 비트 |
| --- | --- |
| 0000 | 11110 |
| 0001 | 01001 |
| 0010 | 10100 |
| ... | ... |

#### 8B/10B (Gigabit Ethernet, PCIe, USB 3.0, SATA)
- 8 비트 → 10 비트
- DC balance + 동기화 보장
- IBM 1983 발명 (Widmer / Franaszek)
- 20% 오버헤드

#### 64B/66B (10 GbE, 40 GbE)
- 64 비트 → 66 비트
- 1.5% 오버헤드 — 효율 ↑
- 10GBASE-R

#### 128B/130B (PCIe 3.0+)
- 1.5% 오버헤드
- 더 효율

### 1.5 B8ZS / HDB3

긴 0 연속 깨는 트릭.

#### B8ZS (Bipolar with 8-Zero Substitution)
- T1 (북미)
- 8 연속 0 → 특정 violation 패턴으로 대체

#### HDB3 (High-Density Bipolar 3-zero)
- E1 (유럽)
- 4 연속 0 → violation 패턴

수신기는 violation 보면 원래 0 으로 복원.

### 1.6 PAM (Pulse Amplitude Modulation)

- 한 심볼이 여러 비트 — 더 빠른 비트율
- **PAM-5** (1000BASE-T): -2, -1, 0, +1, +2 → 2 비트 / 심볼
- **PAM-16** (10GBASE-T): 16 레벨 → 4 비트 / 심볼
- **PAM-4** (400GbE): 4 레벨 → 2 비트 / 심볼

### 1.7 인코딩 비교 표

| 인코딩 | DC | 자기 클록 | 대역폭 효율 | 용도 |
| --- | --- | --- | --- | --- |
| NRZ-L | 큼 | 약 | 1:1 | RS-232 |
| NRZ-I | 줄어듦 | 약 | 1:1 | USB |
| Manchester | 0 | 강 | 1:2 | 10BASE-T |
| Diff Manchester | 0 | 강 | 1:2 | Token Ring |
| 4B/5B + NRZI | 작음 | 중 | 4:5 + 1:1 | 100BASE-TX |
| 8B/10B | 0 | 강 | 8:10 | Gig-E, PCIe |
| 64B/66B | 0 | 강 | 64:66 | 10G+ |
| PAM-5 | 0 | 강 | 8 비트 → 4 심볼 | 1000BASE-T |
| PAM-16 | 0 | 강 | 매우 ↑ | 10GBASE-T |

---

## 2. 디지털 → 아날로그 변조 (Modulation)

매체에 캐리어 (carrier sin 파) 가 있고 비트가 캐리어를 변형.

### 2.1 ASK (Amplitude Shift Keying)

- 진폭 (Amplitude) 으로 비트 표현
- `0 → 작은 진폭, 1 → 큰 진폭` 또는 `0 → 0, 1 → 켜짐` (OOK)
- 잡음에 약함
- 사용: 옛 모뎀, RFID

```
0:  ····__·····
1:  ▔▔▔▔▔▔▔
```

### 2.2 FSK (Frequency Shift Keying)

- 주파수 (Frequency) 로 비트 표현
- `0 → f₁, 1 → f₂`
- ASK 보다 잡음 강함
- 사용: 음성 modem (V.21), Bluetooth (GFSK), LoRa

### 2.3 PSK (Phase Shift Keying)

#### BPSK (Binary PSK)
- 위상 0° / 180° 두 가지
- 1 비트 / 심볼

#### QPSK (Quadrature PSK)
- 위상 4 가지 (0°, 90°, 180°, 270°)
- 2 비트 / 심볼

#### 8-PSK
- 8 위상, 3 비트 / 심볼

위상 차이가 가까울수록 잡음에 약함.

### 2.4 QAM (Quadrature Amplitude Modulation)

진폭 + 위상 동시.

| 차수 | 비트 / 심볼 | 콘스텔레이션 |
| --- | --- | --- |
| 16-QAM | 4 | 16 점 |
| 64-QAM | 6 | 64 점 |
| 256-QAM | 8 | 256 점 |
| 1024-QAM (Wi-Fi 6) | 10 | 1024 점 |
| 4096-QAM (Wi-Fi 7) | 12 | 4096 점 |

차수가 높을수록:
- 비트율 ↑
- 잡음 / 거리에 약함 (점들이 가까워서)
- 송수신 정밀도 요구 ↑

#### 콘스텔레이션 (Constellation)

```
4-QAM (= QPSK):       16-QAM:
                                              
         |  •            •  •  |  •  •
         |               •  •  |  •  •
    •    |    •          ──────┼──────
   ──────┼──────         •  •  |  •  •
         |               •  •  |  •  •
    •    |    •
```

I/Q 평면에서 점의 위치 = 진폭 + 위상.

### 2.5 OFDM (Orthogonal Frequency Division Multiplexing)

- 1 캐리어가 아닌 **수백-수천 부반송파 (subcarrier)** 동시
- 각 부반송파에 QAM 적용
- 다중 경로 페이딩에 강함
- 사용: Wi-Fi (a/g/n/ac), LTE, 5G, DAB

### 2.6 OFDMA (OFDM + Multiple Access)

- OFDM 자원을 여러 사용자에 분할
- 사용: LTE / 5G / Wi-Fi 6+

---

## 3. 아날로그 → 디지털 — PCM (Pulse Code Modulation)

음성을 디지털화하는 표준.

### 3.1 단계

1. **Sampling (샘플링)** — 시간 축 이산화
   - Nyquist 정리: 신호 최대 주파수의 **2 배 이상** 으로 샘플링
   - 전화 음성: 4 kHz × 2 = 8 kHz 샘플
   - CD 오디오: 22 kHz × 2 = 44.1 kHz

2. **Quantization (양자화)** — 진폭 이산화
   - 8 bit = 256 레벨 (전화)
   - 16 bit = 65,536 레벨 (CD)
   - 양자화 잡음 (Quantization Noise) 발생

3. **Encoding (인코딩)** — 비트로 표현

### 3.2 비트율 계산

전화 음성 PCM:
- 8 kHz × 8 bit = **64 kbps** (T1, E1, DS0 의 기본 단위)

### 3.3 압축 — A-law / μ-law

- 작은 신호를 더 정밀히 (로그 압축)
- A-law (유럽), μ-law (미국 / 일본)

### 3.4 PCM 변형

- **DPCM** (Differential PCM) — 차이만 저장
- **ADPCM** — 적응형 DPCM
- **DM** (Delta Modulation) — 1 비트 만

---

## 4. T1 / E1 / SONET (디지털 전송 표준)

| 표준 | 속도 | 채널 (DS0=64K) | 지역 |
| --- | --- | --- | --- |
| **DS0** | 64 kbps | 1 | 음성 1 ch |
| **T1 / DS1** | 1.544 Mbps | 24 | 북미 |
| **E1** | 2.048 Mbps | 30+2 | 유럽 / 한국 |
| **T3 / DS3** | 44.7 Mbps | 672 | 북미 |
| **E3** | 34.4 Mbps | 480 | 유럽 |
| **OC-1 (SONET)** | 51.84 Mbps | — | 광 |
| **OC-3** | 155 Mbps | — | 광 |
| **OC-48** | 2.5 Gbps | — | 광 |
| **OC-192** | 10 Gbps | — | 광 |
| **OC-768** | 40 Gbps | — | 광 |

SONET / SDH 는 광섬유 백본 표준 — 통신사 망.

---

## 5. WDM (Wavelength Division Multiplexing)

광섬유 한 줄에 여러 파장 (= 여러 채널) 동시:
- **CWDM** (Coarse) — 18 채널, 20 nm 간격
- **DWDM** (Dense) — 80+ 채널, 0.4-0.8 nm 간격, 8 Tbps+

데이터센터 / 통신사 백본의 핵심.

---

## 6. 인코딩 / 변조 선택 가이드

```
짧은 케이블 / 컴퓨터 내부 ......... NRZ-L (RS-232)
LAN 동축 / TP, 10 Mbps ............ Manchester (10BASE-T)
Fast / Gig Ethernet ............... 4B/5B+NRZI / PAM-5
10G+ Ethernet ..................... 64B/66B + PAM-4/16
PCIe / USB 3 / SATA ............... 8B/10B → 128B/130B
음성 (전화) ....................... PCM (8 kHz × 8 bit = 64K)
Wi-Fi / LTE / 5G .................. OFDM(A) + QAM
모뎀 (DSL) ........................ QAM
무선 IoT / Bluetooth ............. GFSK / OFDM
광섬유 백본 ....................... OOK + DWDM
```

---

## 7. 함정 / 미세 주제

### 함정 1 — DC balance 무시
NRZ-L 로 트랜스포머 통과 시 동작 X. Manchester / 4B/5B 등 필수.

### 함정 2 — Self-clocking 부족
긴 0 / 1 연속 → 클록 손실. Scrambling / Block code 로 해결.

### 함정 3 — 잡음 환경에 고차수 QAM
SNR 부족하면 1024-QAM 동작 X — 자동 down (16-QAM 등).

### 함정 4 — Nyquist 무시
샘플링 < 신호 최대 주파수 × 2 → Aliasing (별의 마차바퀴 효과).

### 함정 5 — A/D 변환의 비트 부족
음성 8 bit OK, 음악은 16 bit 이상.

### 함정 6 — OFDM 의 PAPR
OFDM 의 큰 Peak-to-Average Power Ratio — 증폭기 비효율.

---

## 8. 학습 자료

- **Data and Computer Communications** (Stallings) — Ch. 5
- **Digital Communications** (Proakis / Salehi)
- **OFDM Wireless Communications** (Andrews)
- Shannon "A Mathematical Theory of Communication" 1948

---

## 9. 관련

- [[transmission-media]] — 매체별 어울리는 인코딩 / 변조
- [[noise-and-errors]] — Shannon / SNR / BER
- [[layer-1-physical]] — 상위
