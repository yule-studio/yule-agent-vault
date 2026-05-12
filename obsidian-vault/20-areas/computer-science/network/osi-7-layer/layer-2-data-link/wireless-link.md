---
title: "무선 L2 — Wi-Fi / Bluetooth / Zigbee / NFC"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T16:20:00+09:00
tags:
  - network
  - layer-2
  - wifi
  - bluetooth
  - zigbee
  - nfc
---

# 무선 L2 — Wi-Fi / Bluetooth / Zigbee / NFC

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 802.11 MAC / Bluetooth / Zigbee / NFC / Thread / LoRa |

**[[layer-2-data-link|↑ L2 Data Link]]** · **[[../osi-7-layer|↑↑ OSI 7]]**

---

# Part A. Wi-Fi (802.11) MAC

## A.1 Wi-Fi 표준 진화 (다시)

| 표준 | 마케팅 | 출시 | 최대 | 주파수 |
| --- | --- | --- | --- | --- |
| 802.11 | — | 1997 | 2 Mbps | 2.4 |
| 802.11a | — | 1999 | 54 Mbps | 5 |
| 802.11b | — | 1999 | 11 Mbps | 2.4 |
| 802.11g | — | 2003 | 54 Mbps | 2.4 |
| 802.11n | Wi-Fi 4 | 2009 | 600 Mbps | 2.4 / 5 |
| 802.11ac | Wi-Fi 5 | 2014 | 3.5 Gbps | 5 |
| 802.11ax | **Wi-Fi 6 / 6E** | 2019/2021 | 9.6 Gbps | 2.4 / 5 / 6 |
| 802.11be | **Wi-Fi 7** | 2024 | 46 Gbps | 2.4 / 5 / 6 (MLO) |

## A.2 Wi-Fi 프레임 종류

| 종류 | 용도 |
| --- | --- |
| **Management** | Beacon, Probe, Auth, Association |
| **Control** | RTS, CTS, ACK, BlockAck |
| **Data** | 실제 페이로드 |

## A.3 802.11 MAC 프레임 구조

```
┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐
│ FC   │ Dur  │ Addr1│ Addr2│ Addr3│ Seq  │ Addr4│ Data │ FCS  │
│  2   │  2   │  6   │  6   │  6   │  2   │  6*  │  var │  4   │
└──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘

* Addr4 는 WDS / Mesh 만
```

### Address 의 의미
- **Addr1** — Receiver
- **Addr2** — Transmitter
- **Addr3** — BSSID 또는 Destination
- **Addr4** — Source (WDS)

→ Ethernet 의 2 개 MAC 보다 복잡 (AP 통과).

## A.4 SSID / BSSID / ESSID

- **SSID** (Service Set ID) — 사용자가 보는 이름 ("Home Wi-Fi")
- **BSSID** — AP 의 MAC 주소 (실제 식별자)
- **ESSID** — 여러 AP 가 같은 SSID 로 (extended)

## A.5 Association 흐름

```
1. Beacon — AP 가 100ms 마다 broadcast (SSID, capability)
2. Probe Request/Response — 능동 스캔
3. Authentication — Open / WEP / 802.1X
4. Association Request/Response — AP 와 결합
5. (WPA/WPA2/WPA3) 4-way handshake — 키 교환
6. 데이터 송수신 가능
```

## A.6 인증 / 보안 진화

| 방식 | 출현 | 상태 |
| --- | --- | --- |
| **Open** | — | 인증 없음 |
| **WEP** (Wired Equivalent Privacy) | 1997 | ❌ 깨짐 (2001) |
| **WPA** (TKIP) | 2003 | 임시, 약함 |
| **WPA2** (AES-CCMP) | 2004 | 표준 |
| **WPA3** (SAE) | 2018 | 모던 |
| **OWE** (Opportunistic Wireless Encryption) | 2019 | Open 의 암호화 버전 |

### WPA3 의 개선
- **SAE** (Simultaneous Authentication of Equals) — 사전 공격 면역
- **PMF** (Protected Management Frames) — Deauth 공격 방어
- 192-bit suite (Enterprise)

### 802.1X / RADIUS (Enterprise)
- EAP (PEAP, EAP-TLS) 로 인증
- 사용자별 다른 키
- 회사 / 학교

## A.7 RTS/CTS / NAV

자세히 → [[csma-cd-ca]]

## A.8 Beamforming / MIMO

- **MIMO** (Multiple Input Multiple Output) — 여러 안테나로 동시 스트림
- **SU-MIMO** — Single User
- **MU-MIMO** — Multi User (Wi-Fi 5 down, Wi-Fi 6 up/down)
- **Beamforming** — 신호를 특정 클라이언트로 집중

## A.9 OFDMA (Wi-Fi 6)

- 한 채널을 여러 사용자에 분할 (RU - Resource Unit)
- 작은 패킷 (IoT, 게임) 효율 ↑
- 지연 ↓

## A.10 MLO (Wi-Fi 7)

- Multi-Link Operation — 2.4 / 5 / 6 GHz 동시
- Aggregation — 대역폭 ↑
- Failover — 한 대역 혼잡 시

## A.11 Wi-Fi 6E vs Wi-Fi 6 vs Wi-Fi 7

| 기준 | Wi-Fi 6 | Wi-Fi 6E | Wi-Fi 7 |
| --- | --- | --- | --- |
| 6 GHz | ❌ | ✅ | ✅ |
| MLO | ❌ | ❌ | ✅ |
| 최대 | 9.6 G | 9.6 G | 46 G |
| QAM | 1024 | 1024 | 4096 |
| 채널 폭 | 160 MHz | 160 MHz | 320 MHz |

---

# Part B. Bluetooth

## B.1 표준 / 버전

| 버전 | 별명 | 속도 | 거리 |
| --- | --- | --- | --- |
| 1.0-2.1 | Classic | 1-3 Mbps | 10m |
| 3.0 | High Speed | 24 Mbps (실제 Wi-Fi 사용) | — |
| 4.0 | **BLE** (Low Energy) | 1 Mbps | 50m |
| 5.0 | — | 2 Mbps | 240m |
| 5.2 | LE Audio | — | — |
| 5.3 / 5.4 | — | — | — |

## B.2 주파수 / 채널

- 2.4 GHz ISM 대역
- 79 채널 (1 MHz, BR/EDR) 또는 40 채널 (2 MHz, BLE)
- **FHSS** (Frequency Hopping) — 1600 hops/sec — Wi-Fi 와 공존

## B.3 Classic Bluetooth — Piconet

```
Master ─── Slave 1
   ├─── Slave 2
   ├─── Slave 3
   └─── ... (최대 7 active + 255 parked)
```

- 1 Master + N Slaves
- 모든 통신은 Master 거침
- Scatternet — 여러 piconet 연결

## B.4 BLE (Bluetooth Low Energy)

- **저전력** — 코인셀 1년+
- IoT / wearable / 위치 추적 (Apple AirTag)
- 작은 페이로드, 짧은 burst

### GATT (Generic Attribute Profile)
- Service > Characteristic > Descriptor
- 클라이언트가 Read / Write / Notify

### Advertising
- BLE 장치가 자기 정보 broadcast
- 30s 이상 광고 (옛) → 200ms 도 (모던)

## B.5 페어링 (Pairing)

| 방식 | 보안 |
| --- | --- |
| **Just Works** | 약함 (이어폰) |
| **Numeric Comparison** | 양쪽 6 자리 비교 |
| **Passkey Entry** | 6 자리 입력 |
| **OOB** (Out-of-Band) | NFC 등으로 키 교환 |

## B.6 LE Audio / LC3 Codec

- Bluetooth 5.2 (2020)
- Multi-stream — 양쪽 이어폰 독립
- Auracast — broadcast (공항 안내 등)
- LC3 — 16% 데이터로 SBC 와 동등 품질

---

# Part C. Zigbee

## C.1 개요

- **IEEE 802.15.4** 기반 L2
- 2.4 GHz / 868 MHz / 915 MHz
- 250 kbps (2.4 GHz)
- 저전력, 메시 네트워크
- IoT 홈 자동화 (Philips Hue, SmartThings)

## C.2 토폴로지

- **Star** — Coordinator + Endpoint
- **Tree** — Coordinator + Router + Endpoint
- **Mesh** — 다중 Router 경로

## C.3 ZHA / Z-Wave / Matter

- **Zigbee** — 802.15.4
- **Z-Wave** — 908 MHz 독점 (Sigma → Silicon Labs)
- **Matter** (2022) — 표준 통합 (Apple / Google / Amazon / Samsung)
  - **Thread** 위 (IPv6 IoT 메시) + Wi-Fi
  - 802.15.4 + IPv6 = 6LoWPAN

---

# Part D. NFC (Near Field Communication)

## D.1 개요

- 13.56 MHz RFID 기반
- **4 cm 이내** (의도적으로 짧음)
- 106-424 kbps
- 보안 결제 / 카드 / 페어링

## D.2 종류

| 종류 | 동작 |
| --- | --- |
| **Type A** | NXP MIFARE |
| **Type B** | ISO 14443 (여권) |
| **Type F** | FeliCa (일본) |
| **Type V** | ISO 15693 (긴 거리, 도서관) |

## D.3 모드

- **Reader/Writer** — 폰이 카드 읽기
- **Card Emulation** — 폰이 카드처럼 (Apple Pay, Samsung Pay)
- **P2P** — 양 NFC 장치 (Android Beam — 사장)

---

# Part E. LoRa / LoRaWAN

## E.1 개요

- **L**ong **Ra**nge — 수 km - 수십 km
- 저속 (0.3 - 50 kbps)
- 저전력 — 배터리 10 년
- **915 MHz (US), 868 MHz (EU), 433 MHz**
- IoT 농업 / 도시 센서

## E.2 LoRaWAN
- LoRa PHY 위의 MAC
- Class A/B/C — 다운링크 시점
- 게이트웨이 → TTN (The Things Network) 등

## E.3 vs NB-IoT
- **NB-IoT** — 셀룰러 (LTE-M)
- 라이센스 주파수 / 유료
- LoRa 는 license-free ISM

---

# Part F. 기타 무선 L2

## F.1 Cellular (LTE / 5G) L2

- **PDCP / RLC / MAC** — 세분된 sub-layer
- 매우 복잡 (3GPP 표준)
- 모바일 핸드오버 / QoS

## F.2 Satellite L2

- 긴 지연 (LEO 30 ms, GEO 600 ms)
- 동기 모드, FEC 강함
- Starlink, OneWeb, Kuiper

## F.3 IrDA (적외선)
- 직진성, 점대점
- 사장 (Bluetooth 가 대체)

## F.4 UWB (Ultra-Wideband)
- 3.1-10.6 GHz
- 정밀 위치 (cm 단위)
- Apple AirTag, Samsung SmartTag+
- 802.15.4z

## F.5 LiFi (Light Fidelity)
- LED 가시광선 변조
- 사무실 / 의료
- Wi-Fi 의 대안 (간섭 X, 보안)

---

## 함정

### 함정 1 — Wi-Fi 의 보안 가정
오픈 Wi-Fi 도청 가능. VPN 필수.

### 함정 2 — BLE 의 페어링 보안
Just Works 는 MITM 가능. Numeric Comparison 권장.

### 함정 3 — NFC 의 "릴레이 공격"
멀리 떨어진 카드의 신호를 중계 → 결제 사기. PIN 동시 검증으로 완화.

### 함정 4 — Zigbee 의 보안 키
공장 출고 키 (Trust Center Link Key) 가 표준. 매뉴얼 확인.

### 함정 5 — LoRa 의 대역폭 과대 기대
50 kbps 면 작은 메시지만. 대용량 X.

### 함정 6 — Wi-Fi 채널 중첩
2.4 GHz 의 다른 채널 사용해도 1/2/3 등은 중첩.

---

## 학습 자료

- IEEE 802.11 / 802.15 / 802.15.4 표준
- **802.11 Wireless Networks: The Definitive Guide** (Matthew Gast)
- **Bluetooth Application Developer's Guide**
- **CWNA Study Guide**
- The Things Network (LoRa)

---

## 관련

- [[csma-cd-ca]] — Wi-Fi 의 CSMA/CA
- [[../layer-1-physical/transmission-media]] — 무선 주파수
- [[layer-2-data-link]] — 상위
