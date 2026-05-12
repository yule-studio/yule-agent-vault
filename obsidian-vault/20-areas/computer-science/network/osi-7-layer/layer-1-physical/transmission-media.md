---
title: "전송 매체 (Transmission Media)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T15:15:00+09:00
tags:
  - network
  - layer-1
  - cable
  - fiber
  - wireless
---

# 전송 매체 (Transmission Media)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | TP / 동축 / 광 / 무선 전체 |

**[[layer-1-physical|↑ L1 Physical]]** · **[[../osi-7-layer|↑↑ OSI 7]]**

---

## 0. 매체의 분류

```
전송 매체
├── 유선 (Guided / Bounded)
│   ├── Twisted Pair  (UTP / STP / S/FTP)
│   ├── 동축 (Coaxial Cable)
│   └── 광섬유 (Optical Fiber) — 싱글모드 / 멀티모드
└── 무선 (Unguided / Unbounded)
    ├── 라디오파  (kHz - GHz)
    ├── 마이크로파  (1 - 300 GHz)
    ├── 적외선 (IR)
    └── 가시광선 (Li-Fi)
```

---

## 1. Twisted Pair (TP) — 꼬임선

### 1.1 왜 꼬는가
두 선을 균일하게 꼬면:
- 외부 EMI 가 양 선에 동일하게 영향 → 차동 신호 (Differential) 로 상쇄
- 자기장 노출이 균형 → 방사 / 수신 모두 줄어듦
- **누화 (Crosstalk)** 감소

쌍마다 꼬임 간격이 다르다 — 인접 쌍 간 누화 더 줄임.

### 1.2 UTP vs STP

| 종류 | 차폐 | 비용 | 속도 | 환경 |
| --- | --- | --- | --- | --- |
| **UTP** (Unshielded) | 없음 | 싸다 | 1-10 Gbps | 일반 사무실 |
| **F/UTP** (Foiled) | 외피 호일 | 중 | 10 Gbps | EMI 보통 환경 |
| **STP** (Shielded) | 각 쌍 차폐 | 비쌈 | 10 Gbps | 산업 / EMI 강함 |
| **S/FTP** | 외피 + 각 쌍 | 매우 비쌈 | 10-40 Gbps | 데이터센터 |

### 1.3 Cat (Category) 등급

| Cat | 주파수 | 속도 (100m) | 표준 |
| --- | --- | --- | --- |
| Cat 3 | 16 MHz | 10 Mbps | 10BASE-T |
| Cat 5 | 100 MHz | 100 Mbps | 100BASE-TX |
| **Cat 5e** | 100 MHz | 1 Gbps | 1000BASE-T |
| **Cat 6** | 250 MHz | 1 Gbps (100m), 10 Gbps (55m) | 1000BASE-T, 10GBASE-T |
| **Cat 6a** | 500 MHz | 10 Gbps (100m) | 10GBASE-T |
| Cat 7 | 600 MHz | 10 Gbps | 차폐 필수 |
| Cat 7a | 1 GHz | 40 Gbps | 차폐 |
| Cat 8 | 2 GHz | 25-40 Gbps (30m) | 데이터센터 |

### 1.4 핀 배치 (RJ-45 / T568B)

```
1  White / Orange
2  Orange
3  White / Green
4  Blue
5  White / Blue
6  Green
7  White / Brown
8  Brown
```

T568A vs T568B 차이는 주황 / 녹색 쌍 swap. 한 케이블 양 끝 같은 표준 (Straight)
또는 다른 표준 (Crossover) — 현대 NIC 는 Auto MDI/MDIX 로 자동 처리.

### 1.5 색깔 / 분류

데이터센터 / 사무실 표준:
- 파란색 — Production
- 빨간색 — Critical / DMZ
- 노란색 — Management
- 회색 — Lab / Test

---

## 2. 동축 케이블 (Coaxial Cable)

### 2.1 구조

```
[중심 도체] — 신호 전송
   |
[유전체] — 폴리에틸렌 등 절연
   |
[외부 도체] — 그물 차폐 (알루미늄 / 구리)
   |
[외피] — 보호
```

내부 / 외부 도체가 같은 축 (coaxial) — EMI 차폐 우수.

### 2.2 종류

| 종류 | 임피던스 | 용도 |
| --- | --- | --- |
| **RG-6** | 75Ω | TV, 케이블 모뎀 |
| **RG-59** | 75Ω | CCTV, 옛 TV |
| **RG-58** | 50Ω | 10BASE2 (Thinnet), 무전기 |
| **RG-8** | 50Ω | 10BASE5 (Thicknet), 무전 |
| **Twin-axial** | 100Ω | 100GbE DAC, IBM SAN |

### 2.3 옛 Ethernet 동축

- **10BASE5 ("Thicknet")** — 노란색 굵은 케이블, 500m, vampire tap. 1980 표준.
- **10BASE2 ("Thinnet")** — 검은 얇은 케이블, 185m, BNC 커넥터.

지금은 거의 사라지고 TP / 광 으로 대체.

### 2.4 케이블 TV / 인터넷

가정 케이블 인터넷 (DOCSIS) — 동축으로 수백 Mbps - 10 Gbps. CMTS 가 동축 → 광
변환.

### 2.5 DAC (Direct Attach Copper)
데이터센터 1-7m 단거리 — 동축 + SFP+ 모듈. 광보다 싸고 지연 ↓.

---

## 3. 광섬유 (Optical Fiber)

### 3.1 구조

```
[Core (코어)] — 굴절률 높음, 빛 통과
[Cladding (클래딩)] — 코어 둘러쌈, 굴절률 낮음 → 전반사
[Buffer / Jacket] — 보호
```

빛은 Core 안에서 **전반사 (Total Internal Reflection)** 로 갇혀 진행.

### 3.2 싱글모드 vs 멀티모드

| 항목 | Single-mode (SMF) | Multi-mode (MMF) |
| --- | --- | --- |
| Core 직경 | 8-10 μm | 50 / 62.5 μm |
| 광원 | Laser (1310/1550 nm) | LED / VCSEL (850/1300 nm) |
| 거리 | 10-100 km | 300-2000m |
| 속도 | 10-400 Gbps+ | 10-100 Gbps |
| 비용 | 모듈 비쌈 | 모듈 싸다 |
| 색 (jacket) | 노란색 | OM3/OM4 = 아쿠아, OM5 = 라임 |

### 3.3 광섬유 표준

| 표준 | Core | 거리 (10G) | 거리 (100G) |
| --- | --- | --- | --- |
| OM1 | 62.5/125 | 33m | — |
| OM2 | 50/125 | 82m | — |
| **OM3** | 50/125 (laser-opt) | 300m | 100m |
| **OM4** | 50/125 | 400m | 150m |
| OM5 | 50/125 (WBMMF) | 400m | 400m |
| **OS1/OS2** (SMF) | 8-10/125 | 10-40 km | 10 km - 80 km |

### 3.4 광 트랜시버

| 폼팩터 | 속도 | 채널 |
| --- | --- | --- |
| **SFP** | 1G | 1 |
| **SFP+** | 10G | 1 |
| **SFP28** | 25G | 1 |
| **QSFP+** | 40G | 4 × 10G |
| **QSFP28** | 100G | 4 × 25G |
| **QSFP-DD** | 400G | 8 × 50G |
| **OSFP** | 800G+ | 8 × 100G |

### 3.5 광 커넥터

| 커넥터 | 특징 |
| --- | --- |
| **LC** | 작음, 데이터센터 표준 |
| **SC** | 큰 사각, 통신사 |
| **ST** | 베이어넷 (옛 표준) |
| **MTP/MPO** | 다중 (12/24 strand), 100G+ |

### 3.6 광섬유 장점

- 거리 — 수십 km 까지 무리피터
- 대역폭 — 단일 섬유 100 Gbps+, WDM 으로 Tbps+
- EMI 면역 — 전기 노이즈 영향 없음
- 도청 어려움 — 보안
- 가벼움 / 작음

### 3.7 광섬유 단점

- 휘어지는 반경 한계 (5-10cm)
- 끝단 dirty connector 손실 (0.5 dB+)
- Splice (접합) 장비 비쌈
- 전기 제공 X — POE 불가

---

## 4. 무선 (Wireless)

### 4.1 주파수 대역

| 대역 | 주파수 | 용도 |
| --- | --- | --- |
| **ELF** | 3-30 Hz | 잠수함 통신 |
| **VLF/LF** | 3-300 kHz | AM 라디오 (장파) |
| **MF** | 300 kHz-3 MHz | AM 라디오 (중파) |
| **HF** | 3-30 MHz | 단파 라디오 |
| **VHF** | 30-300 MHz | FM, TV VHF |
| **UHF** | 300 MHz-3 GHz | TV UHF, GPS, Wi-Fi 2.4G |
| **SHF** | 3-30 GHz | 5 GHz Wi-Fi, 5G sub-6 |
| **EHF** | 30-300 GHz | mmWave 5G, 위성 |

### 4.2 Wi-Fi 주파수 / 대역폭

| 대역 | 채널 폭 | 비고 |
| --- | --- | --- |
| 2.4 GHz | 20 MHz | 13 채널 (비중첩 1/6/11), 멀다, 느림 |
| 5 GHz | 20/40/80/160 MHz | 25+ 채널, 빠름, 짧음 |
| 6 GHz (Wi-Fi 6E) | 20/40/80/160 MHz | Wi-Fi 6E, 59 채널 |

### 4.3 Wi-Fi 표준

| 표준 | 별명 | 속도 (최대) | 주파수 |
| --- | --- | --- | --- |
| 802.11b | — | 11 Mbps | 2.4 |
| 802.11g | — | 54 Mbps | 2.4 |
| 802.11n | Wi-Fi 4 | 600 Mbps | 2.4 / 5 |
| 802.11ac | Wi-Fi 5 | 3.5 Gbps | 5 |
| **802.11ax** | Wi-Fi 6 / 6E | 9.6 Gbps | 2.4 / 5 / 6 |
| **802.11be** | Wi-Fi 7 (2024) | 46 Gbps | 2.4 / 5 / 6 (MLO) |

### 4.4 셀룰러

| 세대 | 출현 | 속도 |
| --- | --- | --- |
| 2G (GSM) | 1991 | 384 Kbps |
| 3G (UMTS) | 2001 | 2 Mbps |
| 4G (LTE) | 2009 | 100 Mbps |
| **5G** | 2019 | 10 Gbps (mmWave) |
| 6G | ~2030 | 100+ Gbps (목표) |

### 4.5 안테나

- **전방향 (Omni)** — 모든 방향 (라우터 안테나)
- **지향성 (Directional)** — Yagi, 위성 디시
- **Isotropic** — 이론적 점원
- **MIMO** — 여러 안테나 (Multiple-In Multiple-Out)
- **Beamforming** — 신호 집중 (Wi-Fi 6, 5G)

### 4.6 전파 특성

- **지상파 (Ground wave)** — < 2 MHz, AM
- **공중파 (Sky wave)** — 2-30 MHz, ionosphere 반사
- **직진파 (Line-of-Sight, LOS)** — > 30 MHz, 마이크로파

### 4.7 다중 접속

- **FDMA** — 주파수 분할
- **TDMA** — 시간 분할
- **CDMA** — 코드 분할 (3G)
- **OFDMA** — 직교 주파수 (4G/5G, Wi-Fi 6)

---

## 5. 매체 비교 종합 표

| 특성 | UTP Cat 6 | 동축 RG-6 | SMF | MMF | Wi-Fi 6 | 5G mmWave |
| --- | --- | --- | --- | --- | --- | --- |
| 속도 | 1-10 Gbps | 1 Gbps (DOCSIS) | 400 Gbps+ | 100 Gbps | 9.6 Gbps | 10 Gbps |
| 거리 | 100m | 수백 m | 80+ km | 400m | 30-50m | 100-300m |
| EMI | 약함 | 강함 | 면역 | 면역 | N/A | N/A |
| 비용 | $ | $ | $$$ | $$ | $$ | $$$$ |
| 설치 | 쉬움 | 쉬움 | 어려움 | 중간 | 매우 쉬움 | 인프라 필요 |
| 보안 | 도청 가능 | 도청 가능 | 어려움 | 어려움 | 도청 가능 | 도청 가능 |

---

## 6. 매체 선택 가이드

```
거리 < 100m + 1G 면 충분 ........... UTP Cat 6
거리 < 100m + 10G ................. UTP Cat 6a / Cat 7
거리 100m - 400m + 10-100G ....... MMF (OM3/OM4) + SR 광
거리 1km+ + 10G+ ................. SMF + LR 광
EMI 강함 ......................... SMF 또는 STP
이동성 ........................... Wi-Fi / 5G
넓은 캠퍼스 ...................... Point-to-Point 무선 / 광
```

---

## 7. 함정

### 함정 1 — UTP 와 STP 혼용 시 차폐 X
한 곳만 차폐 → 안테나 효과로 더 나쁨.

### 함정 2 — Cat 케이블 비교 시 길이 무시
Cat 6 도 100m 면 10G 안 됨 (Cat 6a 필요).

### 함정 3 — 광섬유 mode 혼용
싱글모드 광원 + 멀티모드 케이블 → 동작 X.

### 함정 4 — 광 connector dirty
0.1 dB 손실도 link budget 깨질 수 있음. 청소 + inspection.

### 함정 5 — Wi-Fi 채널 중첩
2.4G 의 1/2/3 채널은 중첩. 1/6/11 만.

### 함정 6 — POE (Power over Ethernet) 용량 무시
Cat 5e 도 POE 가능하지만 발열 문제. 4-pair POE+ 는 Cat 6 권장.

---

## 8. 학습 자료

- **Cabling: The Complete Guide** (Oliviero / Woodward)
- **Optical Networking** (Hill / Sarles)
- TIA/EIA-568 표준
- Cisco "Cabling Best Practices"
- [Wireshark 무선 캡처](https://www.wireshark.org/docs/)

---

## 9. 관련

- [[encoding-modulation]] — 어떤 신호를 보내는가
- [[noise-and-errors]] — 매체에서 발생하는 잡음
- [[cables-connectors-devices]] — 케이블 / 커넥터 / 장치 깊이
- [[layer-1-physical]] — 상위
