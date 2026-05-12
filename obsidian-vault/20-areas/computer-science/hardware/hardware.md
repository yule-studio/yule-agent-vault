---
title: "하드웨어 (Hardware)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags:
  - hardware
  - memory
  - storage
  - transistor
  - logic-gate
---

# 하드웨어 (Hardware)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 수준 |

**[[../computer-science|↑ computer-science]]**

> 이 노트는 **물리 하드웨어** — 트랜지스터부터 SoC 까지. CPU 마이크로아키텍처 /
> ISA 는 [[../computer-architecture/computer-architecture]] 참조.

---

## 1. 한 줄 정의

**컴퓨터의 물리적 구성요소** — 트랜지스터부터 메모리, 저장 장치, 네트워크 카드까지.
소프트웨어가 명령하는 모든 것의 실체.

---

## 2. 역사

| 연도 | 사건 |
| --- | --- |
| 1947 | Bell Labs — 첫 트랜지스터 |
| 1958 | Jack Kilby — IC (Integrated Circuit) |
| 1965 | Gordon Moore — Moore's Law |
| 1968 | Intel 설립 |
| 1970 | DRAM (Intel 1103) |
| 1971 | Intel 4004 (2,300 트랜지스터) |
| 1981 | IBM PC |
| 1991 | NAND Flash (Toshiba 1987 발명) |
| 2007 | iPhone — 모바일 SoC 시대 |
| 2018 | 7nm 양산 (TSMC) |
| 2024 | TSMC 3nm, Apple M4 |

Moore's Law: 18-24 개월마다 트랜지스터 밀도 2 배. 2010 년대 후반부터 둔화.

---

## 3. 디지털 논리 기초

### 3.1 비트 / 바이트

- **bit** — 0 or 1
- **byte** — 8 bit
- **nibble** — 4 bit
- **word** — CPU 단위 (32 / 64 bit)

### 3.2 논리 게이트

| 게이트 | 진리표 |
| --- | --- |
| NOT | A → ¬A |
| AND | A·B |
| OR | A+B |
| XOR | A⊕B |
| NAND | ¬(A·B) — universal |
| NOR | ¬(A+B) — universal |

NAND 또는 NOR 만으로 모든 회로 구성 가능 (universal).

### 3.3 조합 회로 (Combinational)

- **Adder** — Half / Full / Ripple-Carry / Carry-Lookahead
- **Multiplexer (MUX)** — N 입력 → 1 출력
- **Decoder** — N 비트 → 2^N 출력
- **Encoder** — 역
- **ALU** — 산술 + 논리

### 3.4 순차 회로 (Sequential)

- **Flip-Flop** — D, JK, T, SR (1 bit 저장)
- **Latch** — Level-triggered
- **Register** — flip-flop 묶음
- **Counter** — 클록마다 증가
- **State Machine** — Mealy / Moore

### 3.5 클록 (Clock)

- 모든 순차 회로의 박자
- GHz = 10^9 Hz/sec
- Skew, Jitter 가 문제
- PLL (Phase-Locked Loop) 로 분배

---

## 4. 트랜지스터 / 반도체

### 4.1 트랜지스터

- **MOSFET** (Metal-Oxide-Semiconductor FET) — 가장 흔함
- **CMOS** (Complementary MOSFET) — 표준 디지털 회로
- nMOS + pMOS pair

### 4.2 공정 (Process Node)

- "7nm", "5nm", "3nm" — 트랜지스터 게이트 길이
- 작을수록: 빠름, 적은 전력, 작은 칩
- TSMC, Samsung, Intel 이 주요 파운드리

### 4.3 무어의 법칙의 종말

- Dennard Scaling 끝남 (2006)
- 양자 터널링 한계
- 비용 폭증 (EUV)
- → 3D 적층, Chiplet, 새로운 재료 (GaN, SiC)

---

## 5. 메모리 (Memory)

### 5.1 분류

| 종류 | 휘발성 | 속도 | 용도 |
| --- | --- | --- | --- |
| **SRAM** | 휘발 | 빠름 | 캐시 |
| **DRAM** | 휘발 | 중 | 메인 RAM |
| **ROM** | 비휘발 | 느림 | 펌웨어 |
| **EEPROM** | 비휘발 | 느림 | BIOS |
| **NAND Flash** | 비휘발 | 중 | SSD, USB |
| **NOR Flash** | 비휘발 | 빠름 (read) | 펌웨어 |
| **MRAM / FeRAM** | 비휘발 | 빠름 | 신기술 |

### 5.2 SRAM (Static RAM)

- 6 트랜지스터 / bit
- 전원 있는 동안 유지
- 캐시에 사용
- 빠름, 비쌈

### 5.3 DRAM (Dynamic RAM)

- 1 트랜지스터 + 1 캐패시터 / bit
- 캐패시터 누설 → **Refresh** 필요 (수 ms 마다)
- 메인 메모리
- 싸고 밀도 높음

#### DRAM 종류 진화
- SDRAM → DDR → DDR2 → DDR3 → DDR4 → DDR5
- DDR5 (2020) — 4800-6400 MT/s, 32GB DIMM
- **HBM** (High Bandwidth Memory) — 3D 적층, GPU/AI

### 5.4 메모리 컨트롤러

- CPU 내장 (i7, M1 등)
- Channel (듀얼 / 쿼드)
- Bank, Rank, Row, Column

### 5.5 ECC (Error Correcting Code)

- 1 bit 에러 정정, 2 bit 감지
- 서버 / 워크스테이션
- 우주선 비트 플립 / 노화 / Rowhammer 방어

---

## 6. 저장 장치 (Storage)

### 6.1 HDD (Hard Disk Drive)

- 회전 디스크 + 헤드
- 7200 RPM 표준
- Seek time ~5-10ms
- 큰 용량 (20TB+), 싸다
- 데이터 센터 cold storage

### 6.2 SSD (Solid State Drive)

- NAND Flash 기반
- SATA SSD: 500-550 MB/s
- **NVMe** (PCIe 기반): 3-12 GB/s
- 0.1ms 미만 latency

### 6.3 NAND Flash 종류

| 종류 | bit/cell | 수명 (P/E cycles) |
| --- | --- | --- |
| SLC | 1 | 100,000 |
| MLC | 2 | 10,000 |
| TLC | 3 | 3,000 |
| QLC | 4 | 1,000 |

밀도 ↑, 수명 ↓, 가격 ↓.

### 6.4 Wear Leveling

- Flash 셀 수명 제한
- 쓰기를 균등 분배
- TRIM — OS 가 unused 알림
- Over-provisioning

### 6.5 인터페이스

- **SATA** — 6 Gbps
- **SAS** — 서버
- **NVMe** — PCIe 직접
- **M.2** — 폼팩터
- **U.2 / E1.S** — 데이터센터

### 6.6 RAID

[[../operating-system/operating-system#8 파일 시스템]] 참조.

### 6.7 Optane / 3D XPoint
- Intel/Micron — DRAM 과 SSD 사이
- 2022 단종

### 6.8 NVRAM / Persistent Memory
- Battery-backed RAM
- Intel Optane DC PMM (단종)

---

## 7. 버스 / I/O

### 7.1 시스템 버스

- **PCIe** — Peripheral Component Interconnect Express
- **PCIe 5.0** — 32 GT/s per lane, x16 = 64 GB/s
- **PCIe 6.0** — 64 GT/s (2022)
- **CXL** (Compute Express Link) — 메모리 / 가속기 공유

### 7.2 칩셋 / 사우스브리지

- 옛날: Northbridge (메모리) + Southbridge (I/O)
- 지금: CPU 가 메모리 컨트롤러 흡수, 사우스만 남음

### 7.3 인터페이스

| 인터페이스 | 속도 | 용도 |
| --- | --- | --- |
| USB 2.0 | 480 Mbps | 키보드, 마우스 |
| USB 3.x | 5-20 Gbps | 외장 |
| USB4 / Thunderbolt 4 | 40 Gbps | 모니터, 외장 GPU |
| HDMI 2.1 | 48 Gbps | 8K 모니터 |
| DisplayPort 2.0 | 80 Gbps | 모니터 |
| Ethernet | 1G/10G/100G | 네트워크 |

### 7.4 DMA (Direct Memory Access)

- CPU 거치지 않고 장치 ↔ 메모리
- 디스크 / 네트워크 / GPU

### 7.5 Interrupt vs Polling

- **Interrupt** — 이벤트 발생 시 알림
- **Polling** — 주기적 확인
- **NAPI** (Linux) — 둘의 하이브리드 (네트워크)

---

## 8. SoC (System on Chip)

### 8.1 SoC 의 구성

```
CPU + GPU + NPU + DSP + ISP + Modem + Memory Controller + I/O = SoC
```

### 8.2 모바일 SoC

- Apple A / M 시리즈
- Qualcomm Snapdragon
- MediaTek Dimensity
- Samsung Exynos
- Google Tensor

### 8.3 Apple Silicon

- M1 (2020) — ARM, 16 billion 트랜지스터
- M2 / M3 / M4
- Unified Memory Architecture (CPU/GPU 공유)
- Performance + Efficiency core (P-core + E-core)

### 8.4 데이터센터 가속기

- NVIDIA H100 / H200 / Blackwell
- AMD MI300
- Google TPU v5
- Intel Gaudi 3

---

## 9. 디스플레이

### 9.1 패널 종류

- **LCD** — IPS (시야각), VA (대비), TN (속도)
- **OLED** — 자발광, 무한 대비
- **Mini-LED** — LCD + 더 많은 dimming zone
- **Micro-LED** — 차세대

### 9.2 해상도 / 갱신율

- 1080p / 1440p / 4K / 8K
- Refresh rate — 60 / 120 / 144 / 240 Hz
- HDR — Dolby Vision, HDR10+

---

## 10. 입력 장치

### 10.1 키보드

- **Membrane** — 싸다
- **Mechanical** — 스위치 (Cherry MX, Topre)
- **Optical** — 빛 기반 (빠름)

### 10.2 마우스

- **Optical** — LED
- **Laser** — 정밀
- **Polling rate** — 125/500/1000/8000 Hz

### 10.3 터치 / 펜
- 정전식 (Capacitive)
- 압력 감지 (Apple Pencil, Wacom)

---

## 11. 전원 / 쿨링

### 11.1 PSU (Power Supply Unit)

- 80 PLUS 인증 (Bronze ~ Titanium)
- ATX, SFX, SFX-L
- 12VHPWR (RTX 4090)

### 11.2 TDP (Thermal Design Power)

- 발열 ~ 전력 소비
- 데스크톱 CPU: 65-250W
- 서버 CPU: 200-500W
- 데이터센터 GPU: 700W+

### 11.3 쿨링

- **Air** — 히트싱크 + 팬
- **Liquid** — AIO / Custom
- **Immersion** — 데이터센터 (Microsoft 실험)
- **Vapor Chamber** — 모바일

---

## 12. 네트워크 하드웨어

### 12.1 NIC (Network Interface Card)

- 1G / 10G / 25G / 100G / 400G Ethernet
- **DPU** (Data Processing Unit) — NIC + CPU + 가속기 (NVIDIA BlueField)
- **SmartNIC** — 일부 처리 오프로드

### 12.2 스위치 / 라우터

- **L2** — MAC 기반 (Ethernet)
- **L3** — IP 라우팅
- **Spine-Leaf** — 데이터센터

### 12.3 무선

- **Wi-Fi 6 / 6E / 7** — 2024 표준
- **5G / 6G**
- **Bluetooth 5.4**
- **UWB** (Ultra-Wideband) — Apple AirTag

---

## 13. 함정 / 안티패턴

### 함정 1 — SSD 의 무한 쓰기 가정
TBW (Total Bytes Written) 한계.

### 함정 2 — RAM 의 영구성 가정
전원 끊기면 사라짐. fsync.

### 함정 3 — PCIe lane 충돌
M.2 NVMe 사용 시 SATA 비활성 등.

### 함정 4 — TDP = 최대 전력 X
순간 (PL2) 은 훨씬 높음.

### 함정 5 — ECC 없이 서버 운영
1 bit 에러로 데이터 손상.

### 함정 6 — 쿨링 부족
Thermal throttling — 성능 저하.

### 함정 7 — DDR 와 DDR5 모듈 혼용
호환 X — 다른 슬롯.

### 함정 8 — Cosmic ray bit flip
ECC, checksum, replication.

---

## 14. 면접 질문

1. **DRAM vs SRAM**.
2. **HDD vs SSD**.
3. **PCIe lane 동작**.
4. **DMA 가 무엇이고 왜**.
5. **ECC 메모리 필요한 경우**.
6. **NAND Flash wear leveling**.
7. **SoC 구성**.
8. **Moore's Law 종말 의미**.
9. **HBM vs DDR**.
10. **인터럽트 vs 폴링**.

---

## 15. 학습 자료

- **Digital Design and Computer Architecture** (Harris / Harris) — 디지털 로직 + RISC-V
- **The Elements of Computing Systems** (Nisan / Schocken) — NAND2Tetris
- **Computer Organization and Design** (Patterson / Hennessy)
- [Nand to Tetris](https://www.nand2tetris.org/) — 무료 코스
- **AnandTech** — 하드웨어 리뷰
- **WikiChip** — 칩 데이터베이스
- **TechInsights** — 다이 분석

---

## 16. 관련

- [[../computer-architecture/computer-architecture]] — CPU 마이크로아키텍처
- [[../operating-system/operating-system]] — 메모리, 디스크
- [[../network/network]] — 네트워크 하드웨어
- [[../computer-science|↑ computer-science]]
