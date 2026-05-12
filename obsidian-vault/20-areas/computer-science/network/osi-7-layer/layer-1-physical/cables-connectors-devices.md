---
title: "케이블 / 커넥터 / L1 장치"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T15:30:00+09:00
tags:
  - network
  - layer-1
  - cable
  - connector
  - nic
---

# 케이블 / 커넥터 / L1 장치

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | RJ-45 / BNC / 광 / 트랜시버 / NIC / Hub / Modem |

**[[layer-1-physical|↑ L1 Physical]]** · **[[../osi-7-layer|↑↑ OSI 7]]**

---

## 1. 커넥터 (Connectors)

### 1.1 구리 (Copper)

| 커넥터 | 핀 / 컨택 | 용도 |
| --- | --- | --- |
| **RJ-45** | 8P8C | Ethernet (UTP) — 가장 흔함 |
| **RJ-11** | 4P / 6P | 전화선 |
| **BNC** | 베이어넷 | 10BASE2, CCTV, 오실로스코프 |
| **F-type** | 나사 | TV 동축 (RG-6) |
| **N-type** | 나사 | RF, 위성 |
| **SMA** | 나사 | RF, Wi-Fi 안테나 |
| **DB-9 (DE-9)** | 9 핀 | RS-232 |
| **DB-25** | 25 핀 | RS-232 (옛) |
| **RJ-48** | 8P8C | T1 (E1) |

### 1.2 광섬유 (Fiber)

| 커넥터 | 특징 | 용도 |
| --- | --- | --- |
| **LC** | 작은 사각, 래치 | 데이터센터 표준 |
| **SC** | 큰 사각, 푸시풀 | 통신사, 공공망 |
| **ST** | 베이어넷 | 옛 표준 |
| **FC** | 나사 | 통신 측정 |
| **MTP / MPO** | 다중 (12/24) | 100G+ 백본 |
| **MTRJ** | 듀얼 LC 변형 | 거의 사장 |

### 1.3 광 연마 종류 (Polish)

| 종류 | 표시 | Return Loss |
| --- | --- | --- |
| PC | 검은 | -40 dB |
| **UPC** (Ultra) | 파란 | -50 dB |
| **APC** (Angled) | 녹색 (8°) | -65 dB |

APC 는 끝면 8° 각도 — 반사파 흡수 → 고성능. CATV, PON 에 필수.

⚠️ APC ↔ UPC 직접 결합 X — 손실 / 손상.

---

## 2. 트랜시버 / 모듈 (Transceiver)

### 2.1 SFP / SFP+ 패밀리

| 폼팩터 | 속도 | 비고 |
| --- | --- | --- |
| **SFP** | 1G | 핫 스왑, 1990s 후기 |
| **SFP+** | 10G | 데이터센터 표준 (10GbE) |
| **SFP28** | 25G | 25/50 GbE |
| **SFP56** | 50G | PAM-4 |
| **QSFP+** | 40G | 4 lane × 10G |
| **QSFP28** | 100G | 4 × 25G |
| **QSFP56** | 200G | 4 × 50G PAM-4 |
| **QSFP-DD** | 400G | 8 × 50G PAM-4 |
| **OSFP** | 800G+ | 8 × 100G |
| **CFP** / **CFP2** | 100G | 옛 100G, 큰 폼팩터 |

### 2.2 광 트랜시버 종류 (10GBASE-)

| 표준 | 매체 | 거리 | 파장 |
| --- | --- | --- | --- |
| **10GBASE-SR** | MMF OM3 | 300m | 850 nm |
| **10GBASE-LR** | SMF | 10 km | 1310 nm |
| **10GBASE-ER** | SMF | 40 km | 1550 nm |
| **10GBASE-ZR** | SMF | 80 km | 1550 nm |
| **10GBASE-T** | UTP Cat 6a | 100m | — (전기) |
| **10GBASE-CR** (DAC) | 동축 | 1-7m | — (전기) |

100G / 400G 도 같은 패턴 (SR4 / LR4 / ER4 / ZR4).

### 2.3 DAC vs AOC

- **DAC (Direct Attach Cable)** — 트랜시버 일체형 동축, 1-7m, 싸다
- **AOC (Active Optical Cable)** — 트랜시버 일체형 광, 3-100m, 가볍다

데이터센터 ToR (Top of Rack) ↔ 서버 짧은 거리에 흔히 사용.

### 2.4 Vendor lock-in

Cisco / Juniper / Arista 등이 자사 호환 SFP 만 받음 — 3rd party 는 코딩 필요.
가격 차이 5-10 배. "코딩" 서비스로 호환 가능.

---

## 3. NIC (Network Interface Card)

### 3.1 구성

```
┌──────────────────────────────────┐
│   PHY     │   MAC   │   Buffer  │
│  (L1)     │   (L2)  │   / DMA   │
│  Twisted  │ Ethernet│           │
│  Pair     │ Frame   │           │
│  Optical  │ CSMA/CD │           │
└──────────────────────────────────┘
            ↑ PCIe / USB ↑
                Host
```

- **PHY** — 신호 변환 (L1)
- **MAC** — 프레임 / CSMA (L2)
- **Buffer** — RX/TX ring
- **DMA Engine** — 호스트 메모리 직접 접근

### 3.2 인터페이스

| 인터페이스 | 속도 |
| --- | --- |
| **PCIe x1 Gen3** | 1 GB/s |
| **PCIe x4 Gen3** | 4 GB/s |
| **PCIe x8 Gen3** | 8 GB/s |
| **PCIe x16 Gen4** | 32 GB/s |

10 GbE NIC = PCIe x4 Gen3 / 25-100 GbE = PCIe x8/x16.

### 3.3 NIC 의 주요 기능

- **Checksum offload** — TCP/IP/UDP 체크섬 HW 계산
- **TSO** (TCP Segmentation Offload) — 큰 페이로드를 NIC 가 잘라서 보냄
- **LRO** (Large Receive Offload) — 수신 시 합쳐서 OS 에 전달
- **RSS** (Receive Side Scaling) — 여러 CPU 코어로 분산
- **SR-IOV** — VM 직접 NIC
- **DCB / FCoE** — Storage over Ethernet
- **PTP** (Precision Time Protocol) — HW 타임스탬프

### 3.4 SmartNIC / DPU

- NIC + ARM CPU + 가속기
- 패킷 처리 / 보안 / 가상화 오프로드
- **NVIDIA BlueField**, **Intel IPU**, AWS Nitro

---

## 4. 허브 (Hub)

### 4.1 동작

- Multi-port 리피터
- 한 포트 입력 → 모든 포트로 broadcast
- **하나의 충돌 도메인** + **하나의 broadcast 도메인**
- Half-duplex 강제

### 4.2 종류

- **Passive Hub** — 단순 분기
- **Active Hub** — 신호 증폭

### 4.3 왜 사라졌는가

- 충돌 / 효율 저하
- 보안 약함 (모두 들음)
- L2 Switch 가 가격 / 성능 압도

지금 "Hub" 라 부르는 건 보통 Switch.

---

## 5. 리피터 (Repeater)

- 감쇠된 신호 재생성 — 이더넷 5-4-3 규칙 (5 세그먼트, 4 리피터, 3 active)
- 광 리피터 — 광 → 전기 → 광
- 광 증폭기 (EDFA, Raman) — 광 그대로

---

## 6. 모뎀 (Modem)

### 6.1 정의
**Mo**dulator + **Dem**odulator. 디지털 ↔ 아날로그.

### 6.2 종류

| 종류 | 매체 | 속도 |
| --- | --- | --- |
| **Dial-up modem** | 전화선 (RJ-11) | 56 kbps (V.92) |
| **DSL modem** | 전화선 (xDSL) | 1-100 Mbps |
| **Cable modem** | 동축 (DOCSIS) | 100M - 10 Gbps |
| **Optical modem (ONT/ONU)** | GPON / EPON | 1-10 Gbps |
| **Cellular modem** | LTE / 5G | 100M - 10 Gbps |
| **Satellite modem** | 위성 | 100M - 500 Mbps (Starlink) |

### 6.3 DSL 종류

- **ADSL** (Asymmetric) — 다운 빠름 / 업 느림
- **VDSL** (Very High) — ADSL 보다 빠름, 거리 짧음
- **G.fast** — 1 Gbps, 100m

---

## 7. NIC 의 LED 의미

| LED | 의미 |
| --- | --- |
| **Link** (녹색) | 연결 OK |
| **Activity** (깜빡) | 패킷 송수신 |
| **Speed** (색 변화) | 10/100/1000 |
| **Collision** | 충돌 (Half-duplex 시) |
| **FDX** | Full Duplex |

---

## 8. 케이블 진단 도구

| 도구 | 용도 |
| --- | --- |
| **Cable Tester** | UTP 핀 / 단선 / wire map |
| **TDR** (Time Domain Reflectometer) | 단선 위치 |
| **OTDR** (Optical TDR) | 광 단선 / 손실 위치 |
| **OPM** (Optical Power Meter) | 광 출력 측정 |
| **Inspection Microscope** | 광 connector 청결 확인 |
| **Fluke / DSX** | 인증 측정 (Cat 등급 검증) |

---

## 9. POE (Power over Ethernet)

### 9.1 표준

| 표준 | 전력 | 페어 |
| --- | --- | --- |
| IEEE 802.3af | 15.4 W | 2 페어 |
| **IEEE 802.3at (POE+)** | 30 W | 2 페어 |
| **IEEE 802.3bt Type 3 (POE++)** | 60 W | 4 페어 |
| **IEEE 802.3bt Type 4** | 90-100 W | 4 페어 |

### 9.2 용도
- IP 카메라, VoIP 전화
- Wi-Fi AP
- LED 조명 (PoE Lighting)
- 노트북 충전 (90W)

### 9.3 함정
- Cat 5e 도 가능하지만 발열 — Cat 6 권장
- Cat 케이블 길어질수록 전압 강하 — 100m 한계
- Switch / Injector 의 총 전력 (Budget) 확인

---

## 10. 함정

### 함정 1 — Crossover 케이블 vs Straight
현대는 Auto MDI/MDIX — 신경 안 써도 됨. 옛 장비는 Cross.

### 함정 2 — SFP 호환성
Brand-locked. 호환 SFP 사용 시 "service unsupported-transceiver" 명령.

### 함정 3 — 광 모듈 hot-plug
지원하지만 빠르게 — 일부 OS 는 link flap.

### 함정 4 — 광 청결 무시
가장 흔한 광 문제. 손가락 한 번이 -1 dB 손실.

### 함정 5 — DAC 길이 한계
1-7m 만. 그 이상은 AOC 또는 광.

### 함정 6 — PoE 와 비 PoE 혼용
구형 비 PoE 장비에 PoE 주면 손상 위험. PoE Detection 거의 표준이지만 확인.

### 함정 7 — NIC 의 Half-duplex 강제
Auto-negotiate 실패 → 한쪽 Full / 한쪽 Half — CRC 폭증.

---

## 11. 학습 자료

- **Cabling: The Complete Guide** (Andrew Oliviero)
- **Fiber Optic Networks** (Paul Green)
- TIA-568 / TIA-606 표준
- Cisco "10 Gigabit Ethernet Deployment Guide"
- [FS.com](https://www.fs.com/) — 제품 가이드

---

## 12. 관련

- [[transmission-media]] — 매체 깊이
- [[encoding-modulation]] — 트랜시버 안의 인코딩
- [[../layer-2-data-link/layer-2-data-link]] — NIC 의 MAC 부분
- [[layer-1-physical]] — 상위
