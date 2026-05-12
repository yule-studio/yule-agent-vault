---
title: "잡음 / 오류 / Shannon 한계"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T15:25:00+09:00
tags:
  - network
  - layer-1
  - noise
  - ber
  - shannon
---

# 잡음 / 오류 / Shannon 한계

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 신호 감쇠 / 지연 / 잡음 4 종 / Shannon |

**[[layer-1-physical|↑ L1 Physical]]** · **[[../osi-7-layer|↑↑ OSI 7]]**

---

## 1. 신호의 3 가지 적 (Impairment)

매체를 통과하면서 신호는 다음 3 가지로 열화:

1. **감쇠 (Attenuation)** — 진폭 감소
2. **지연 / 왜곡 (Delay distortion)** — 주파수별 다른 속도
3. **잡음 (Noise)** — 신호 + 원치 않는 신호

---

## 2. 감쇠 (Attenuation)

### 2.1 정의
- 매체를 따라 진행하며 **신호 세기 감소**
- 단위: dB (데시벨) — 로그 척도
- `Loss (dB) = 10 × log₁₀(P_in / P_out)`

### 2.2 매체별 감쇠

| 매체 | 감쇠 |
| --- | --- |
| UTP Cat 6 | 약 20 dB / 100m @ 100 MHz |
| 동축 RG-6 | 약 6 dB / 100m @ 100 MHz |
| SMF (1550 nm) | 약 0.2 dB / km |
| MMF (850 nm) | 약 3 dB / km |
| 무선 (자유공간) | 거리² 에 비례 (Free Space Path Loss) |

### 2.3 주파수 의존성
- 높은 주파수가 더 감쇠 — Cat 케이블 사양이 주파수별
- → 이퀄라이저 (Equalizer) 로 보정

### 2.4 보상 방법
- **리피터 (Repeater)** — 디지털 재생성
- **증폭기 (Amplifier)** — 아날로그 증폭 (잡음도 같이)
- **이퀄라이저** — 주파수별 보정
- **광섬유 EDFA** (Erbium-Doped Fiber Amplifier) — 광 직접 증폭

---

## 3. 지연 / 왜곡 (Delay Distortion)

### 3.1 정의
- 주파수마다 매체에서 **다른 속도** 로 전파
- → 같은 시점에 보낸 신호가 다른 시점에 도착 → 펄스 퍼짐 → **ISI** (Inter-Symbol Interference)

### 3.2 분산 (Dispersion) — 광섬유

#### Chromatic Dispersion
- 파장별 다른 속도 → 펄스 퍼짐
- 단일파장 (DFB Laser) + 분산 보상 섬유 (DCF) 로 해결

#### Modal Dispersion (MMF)
- 빛의 여러 모드가 다른 경로
- → MMF 의 거리 / 속도 제한

#### Polarization Mode Dispersion (PMD)
- 편광에 따른 속도 차
- 100G+ 에서 문제

### 3.3 ISI 방지
- 심볼 간격 충분히
- 펄스 모양 (Raised Cosine, Pulse Shaping)
- 이퀄라이저
- OFDM (cyclic prefix)

---

## 4. 잡음 (Noise) — 4 종

### 4.1 열잡음 (Thermal Noise / Johnson-Nyquist Noise)

- **전도체 내 전자의 열 운동** → 무작위 신호
- 절대 영도 제외 어디든 존재 — 기본 noise floor
- `N = k × T × B`
  - k: Boltzmann 상수
  - T: 절대 온도
  - B: 대역폭
- 대역폭 ↑ → 잡음 ↑

### 4.2 혼변조 왜곡 (Intermodulation Distortion)

- 두 주파수 f₁, f₂ 가 비선형 증폭기 통과 → **f₁ ± f₂, 2f₁ - f₂** 등 새 주파수 생성
- 무전기 / 셀룰러에서 큰 문제

### 4.3 누화 (Crosstalk)

- 인접 케이블 / 회로 간 상호 간섭
- 종류:
  - **NEXT** (Near-End Crosstalk) — 같은 쪽
  - **FEXT** (Far-End Crosstalk) — 반대편
  - **AXT** (Alien Crosstalk) — 다른 케이블에서

방어: TP 꼬임, 차폐 (STP), 거리.

### 4.4 충격성 잡음 (Impulse Noise)

- 짧고 강한 펄스 — 번개, 모터 시동, 전원
- 디지털 통신의 주범 — burst error

---

## 5. SNR (Signal-to-Noise Ratio)

### 5.1 정의

```
SNR = P_signal / P_noise
SNR (dB) = 10 × log₁₀(P_signal / P_noise)
```

### 5.2 일반적 SNR

| 환경 | SNR |
| --- | --- |
| 좋은 Wi-Fi 연결 | 40+ dB |
| 보통 Wi-Fi | 20-25 dB |
| 나쁜 Wi-Fi (작동 한계) | 10-15 dB |
| TV 방송 | 30+ dB |
| 오디오 CD | 96 dB |

### 5.3 SNR 과 QAM 차수

| SNR (dB) | 최대 QAM |
| --- | --- |
| 8 | QPSK |
| 15 | 16-QAM |
| 22 | 64-QAM |
| 28 | 256-QAM |
| 34 | 1024-QAM (Wi-Fi 6) |
| 40 | 4096-QAM (Wi-Fi 7) |

이래서 Wi-Fi 가 거리에 따라 속도 자동 down (Rate Adaptation).

---

## 6. Eb/N0

### 6.1 정의

- **비트당 에너지** / **잡음 PSD (Power Spectral Density)**
- `Eb/N0 = (S/N) × (B/R)`
  - B: 대역폭
  - R: 비트율
- 디지털 변조 비교의 정규화 표준

### 6.2 BER 그래프

각 변조 방식마다 BER vs Eb/N0 곡선 — 같은 BER 위해 필요한 Eb/N0:
- BPSK: ~ 9.6 dB
- QPSK: ~ 9.6 dB (BPSK 와 같음)
- 16-QAM: ~ 13.4 dB
- 64-QAM: ~ 17.6 dB
- 256-QAM: ~ 23.5 dB

(BER = 10⁻⁵ 기준)

---

## 7. BER (Bit Error Rate)

### 7.1 정의
오류 비트 / 전송 비트.

### 7.2 시스템별 요구

| 시스템 | BER 요구 |
| --- | --- |
| Ethernet | 10⁻¹² |
| Wi-Fi | 10⁻⁵ - 10⁻⁶ |
| 5G | 10⁻⁶ |
| 음성 통화 | 10⁻³ - 10⁻⁴ |
| HDTV | 10⁻¹⁰ |
| 위성 (이전) | 10⁻⁷ |

### 7.3 BER 측정
- BERT (Bit Error Rate Tester) — 알려진 패턴 송수신
- PRBS (Pseudo-Random Bit Sequence) — 2³¹-1 등

---

## 8. Shannon 정리 (Shannon-Hartley)

### 8.1 정리

**잡음이 있는 채널의 최대 정보 전송률**:

```
C = B × log₂(1 + S/N)
```

- C: 채널 용량 (bps)
- B: 대역폭 (Hz)
- S/N: SNR (선형, dB 아님)

**Claude Shannon 1948** — 정보 이론의 토대.

### 8.2 예시

전화 회선:
- B = 3 kHz, SNR = 30 dB (= 1000)
- C = 3000 × log₂(1001) ≈ 30 kbps

→ 56 kbps 모뎀이 회선 한계 거의 끝까지 짠 결과.

Wi-Fi (Wi-Fi 6, 5 GHz, 160 MHz):
- B = 160 MHz, SNR = 30 dB
- C ≈ 160e6 × log₂(1001) ≈ 1.6 Gbps (per spatial stream)
- 8 stream MIMO → 9.6 Gbps

### 8.3 Shannon 한계의 함의

- 정보 전송률을 무한히 늘리려면 대역폭 또는 SNR 증가
- 둘 다 한계 → 5G mmWave, 광섬유 WDM 등 새 매체
- 압축 / 코딩 (Turbo Code, LDPC, Polar Code) 으로 한계에 점점 가까워짐

---

## 9. Nyquist 정리 (잡음 없는 경우)

```
C = 2 × B × log₂(L)
```

- L: 신호 레벨 수
- 잡음 없을 때 이상적 최대

QAM-1024 → L=32 per dimension → 더 많은 비트 / 심볼.

---

## 10. 오류 정정 (FEC)

### 10.1 ARQ (Automatic Repeat reQuest)
- 오류 감지 → 재전송 요청
- L2 / L4 에서

### 10.2 FEC (Forward Error Correction)
- 송신 시 redundancy 추가 → 수신에서 직접 정정
- 재전송 불가 / 지연 큰 채널 (위성, 5G, 광)

### 10.3 주요 FEC

| 코드 | 특징 |
| --- | --- |
| **Hamming** | 1 비트 정정 |
| **Reed-Solomon** | 버스트 오류 강함 — CD, DVD, QR |
| **Turbo Code** | 3G/4G, Shannon 근접 |
| **LDPC** (Low-Density Parity-Check) | 5G, Wi-Fi 6, DVB-S2 |
| **Polar Code** | 5G 제어 채널, Shannon 달성 가능 |

### 10.4 인터리빙 (Interleaving)
- 데이터 순서 섞기 → 버스트 오류를 분산 → FEC 가 처리하기 쉬움
- 사용: CD (스크래치 견딤), Wi-Fi, LTE

---

## 11. 오류 검출

### 11.1 Parity Bit
- 1 비트 — 짝수 / 홀수
- 단점: 짝수 비트 오류 못 잡음

### 11.2 Checksum
- 합산 후 1's complement
- TCP / UDP / IP 사용 (16 bit)
- 약한 — 일부 swap 못 잡음

### 11.3 CRC (Cyclic Redundancy Check)
- 다항식 나눗셈
- CRC-16, CRC-32 (Ethernet), CRC-CCITT
- 버스트 오류 강함
- 자세히 → [[../layer-2-data-link/error-detection-crc]]

---

## 12. 함정 / 미세 주제

### 함정 1 — SNR 의 단위 혼동
SNR 선형 비 vs dB. Shannon 식에서는 선형.

### 함정 2 — Nyquist 와 Shannon 혼동
- Nyquist: 잡음 없는 채널의 이론 최대
- Shannon: 잡음 있는 채널의 이론 최대

### 함정 3 — BER vs FER (Frame Error Rate)
다른 단위 — 한 frame 에 여러 비트.

### 함정 4 — 잡음 무관 케이블 신뢰
완벽한 케이블은 없음 — 항상 BER 측정.

### 함정 5 — Eb/N0 vs SNR
다른 정규화 — 변조 비교는 Eb/N0.

### 함정 6 — Free Space Path Loss 가정
실제 무선은 다중 경로 / 페이딩 / 흡수 — 훨씬 큰 손실.

---

## 13. 학습 자료

- **Data and Computer Communications** (Stallings) — Ch. 3, 5
- **Digital Communications** (Proakis)
- **Principles of Digital Communication** (Robert Gallager) — MIT OCW
- Shannon "A Mathematical Theory of Communication" 1948 — 원논문
- **Modern Wireless Communications** (Haykin / Moher)

---

## 14. 관련

- [[encoding-modulation]] — 잡음에 강한 변조 선택
- [[transmission-media]] — 매체별 잡음 특성
- [[../layer-2-data-link/error-detection-crc]] — L2 의 CRC
- [[layer-1-physical]] — 상위
