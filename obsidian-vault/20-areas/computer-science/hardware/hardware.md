---
title: "하드웨어 (Hardware) — Hub"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags:
  - hardware
  - hub
  - memory
  - storage
  - transistor
  - soc
---

# 하드웨어 (Hardware) — Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 수준 단일 노트 |
| v.1.1.0 | 2026-05-13 | engineering-agent/tech-lead | 책 수준 두텁게 재작성 |
| v.2.0.0 | 2026-05-13 | engineering-agent/tech-lead | network hub 패턴 적용 — hub + 10 토픽 폴더 + 깊이 파일로 분해 |

**[[../computer-science|↑ computer-science]]**

> 이 노트는 **물리 하드웨어 전체의 hub**. 각 토픽은 별도 폴더 / 노트로 깊이 다룬다.
> CPU 마이크로아키텍처·ISA·파이프라인은 [[../computer-architecture/computer-architecture]].
> OS 관점 메모리·스케줄러는 [[../operating-system/operating-system]].
> 네트워크 프로토콜 레이어는 [[../network/network]].

---

## 0. 하드웨어가 다루는 영역

전자가 흘러서 0/1 을 의미하게 만드는 트랜지스터부터, 그것을 모아 만든 메모리 / 저장 / 통신 / SoC 까지. 소프트웨어가 "명령" 하는 모든 것의 실체.

읽는 순서 (물리 → 시스템):

1. [[digital-logic/digital-logic|디지털 논리]] — bit / 게이트 / 클록
2. [[transistor/transistor|트랜지스터·반도체]] — MOSFET / 공정 / 패키징
3. [[memory/memory|메모리]] — SRAM / DRAM / DDR / HBM / ECC
4. [[storage/storage|저장 장치]] — HDD / SSD / NAND / RAID / 인터페이스
5. [[bus-io/bus-io|버스·I/O]] — PCIe / USB / DMA / 인터럽트
6. [[soc/soc|SoC]] — 모바일 SoC / Apple Silicon / 데이터센터 가속기
7. [[display/display|디스플레이]] — LCD / OLED / 갱신율 / HDR
8. [[input/input|입력 장치]] — 키보드 / 마우스 / 터치
9. [[power-cooling/power-cooling|전원·쿨링]] — PSU / VRM / TDP / 쿨링
10. [[network-hardware/network-hardware|네트워크 하드웨어]] — NIC / DPU / 스위치 / RDMA

---

## 1. 토픽별 핵심 + 깊이 노트

### 1.1 [[digital-logic/digital-logic|디지털 논리]]
bit / byte / word, NAND universality, 조합 회로 (Adder / MUX / Decoder), 순차 회로 (Latch / FF / FSM), 클록·setup/hold/skew, HDL.
- [[digital-logic/logic-gates|논리 게이트와 부울 대수]]
- [[digital-logic/combinational-circuits|조합 회로]]
- [[digital-logic/sequential-circuits-clock|순차 회로와 클록]]

### 1.2 [[transistor/transistor|트랜지스터·반도체]]
MOSFET / CMOS, planar → FinFET → GAA, EUV, 누설 전류, 동적 전력 식, 공정 노드 표기, 첨단 패키징.
- [[transistor/mosfet-cmos|MOSFET·CMOS 인버터]]
- [[transistor/process-node-evolution|공정 노드 진화]]

### 1.3 [[memory/memory|메모리]]
계층 / latency / bandwidth, SRAM 6-T, DRAM 1T1C + refresh, DDR1~DDR6 진화, HBM 3D 적층, CXL.mem, ECC + Rowhammer.
- [[memory/memory-hierarchy|메모리 계층]]
- [[memory/dram-cell|DRAM 셀 동작]]
- [[memory/ddr-evolution|DDR 세대 진화]]
- [[memory/ecc-rowhammer|ECC 와 Rowhammer]]

### 1.4 [[storage/storage|저장 장치]]
HDD 기계 구조, SSD 컨트롤러 / FTL / wear leveling / TRIM, NAND 타입 (SLC~QLC), RAID 0/1/5/6/10, SATA / SAS / NVMe / U.2 / M.2 / EDSFF.
- [[storage/hdd|HDD]]
- [[storage/ssd-and-ftl|SSD 와 FTL]]
- [[storage/raid-levels|RAID 레벨]]
- [[storage/storage-interfaces|스토리지 인터페이스]]

### 1.5 [[bus-io/bus-io|버스·I/O]]
PCIe 1.0–7.0 lane×gen, CXL, USB / Thunderbolt 진화, DMA + IOMMU, MSI / MSI-X, NAPI.
- [[bus-io/pcie|PCIe]]
- [[bus-io/dma-iommu|DMA 와 IOMMU]]
- [[bus-io/usb-thunderbolt|USB·Thunderbolt]]

### 1.6 [[soc/soc|SoC]]
SoC 구성 (CPU + GPU + NPU + DSP + ISP + Codec + Modem), Apple Silicon UMA, 모바일 SoC 비교, 데이터센터 가속기 (H100 / MI300 / TPU / Cerebras), NVLink / Infinity Fabric / NVSwitch.
- [[soc/soc-anatomy|SoC 구성 요소]]
- [[soc/datacenter-accelerators|데이터센터 가속기]]

### 1.7 [[display/display|디스플레이]]
TN / IPS / VA / Mini-LED / OLED / QD-OLED / Micro-LED, 해상도 / 갱신율, VRR / G-Sync / FreeSync, HDR (HDR10 / Dolby Vision).
- [[display/panel-types|패널 기술]]

### 1.8 [[input/input|입력 장치]]
키보드 스위치 (Membrane / Mechanical / Optical / Hall / Topre), 마우스 센서 / DPI / 폴링, 터치 (저항 / 정전), 펜 / 디지타이저.
- [[input/keyboard-switches|키보드 스위치]]

### 1.9 [[power-cooling/power-cooling|전원·쿨링]]
PSU 80 PLUS / 12V-2x6, VRM phase, TDP / PL1 / PL2 / Tau, 공랭 / 수랭 / immersion, PUE / WUE.
- [[power-cooling/psu-vrm|PSU 와 VRM]]
- [[power-cooling/tdp-power|TDP 와 전력]]

### 1.10 [[network-hardware/network-hardware|네트워크 하드웨어]]
NIC (RSS / TSO / GRO / RDMA), DPU / SmartNIC (BlueField / AWS Nitro), L2/L3 스위치, Spine-Leaf / Dragonfly, 광 트랜시버 / PAM4 / CPO, Wi-Fi 6/6E/7 / Bluetooth 5.4 / UWB.
- [[network-hardware/nic-offload|NIC 가속 기능]]
- [[network-hardware/switch-fabric|스위치·패브릭]]
- [[network-hardware/rdma-infiniband|RDMA·InfiniBand]]

---

## 2. 모든 토픽이 공유하는 단위 / 식

- **시간**: ns / μs / ms — 캐시는 ns, NVMe 는 μs, HDD 는 ms.
- **데이터**: 1 KB = 10³, 1 KiB = 2¹⁰. 모호하면 KiB/MiB/GiB.
- **주파수**: Hz, MT/s. `bandwidth(GB/s) = MT/s × bus_width(byte) / 1000`.
- **전력**: `P_dynamic = α·C·V²·f`. DVFS 의 본질은 V² 의존성.
- **TDP** ≠ 최대 전력. PL2 / Tau 가 짧은 부스트 전력.

---

## 3. 함정 / 안티패턴 (모든 토픽 공통)

전 토픽에 적용되는 함정만 모음. 각 토픽별 상세 함정은 해당 hub 의 §끝에.

1. **TDP 가 최대 전력이라는 가정** — PL2 의 1.5~2× 가 흐를 수 있다.
2. **DRAM 의 영구성 가정** — 전원 끊기면 모두 휘발. `fsync()` 누락이 가장 흔한 데이터 손실 원인.
3. **PCIe lane 충돌** — M.2 NVMe 가 SATA 포트 비활성시키는 보드 흔함. 매뉴얼 lane 표 확인.
4. **ECC 없이 서버 운영** — 1 bit 비트 플립으로 DB row 손상.
5. **Cosmic ray / Rowhammer** — 미션 크리티컬 서비스는 ECC + checksum + replication + 정기 scrub.
6. **NUMA 무시** — 듀얼 소켓 서버에서 메모리·NIC 가 한 소켓에 묶이면 cross-socket 패널티 2 배.
7. **Bandwidth 와 Latency 혼동** — 100 G 라인이라도 단일 RPC latency 는 라인 속도 무관.
8. **USB-C 케이블 모두 동일하다는 가정** — 같은 모양이라도 480 Mbps ~ 120 Gbps. 전력도 5–240 W.
9. **RAID 5 + 큰 HDD** — rebuild 동안 두 번째 디스크 fail 위험 큼 → RAID 6 / 10.
10. **SR-IOV / IOMMU 비활성 채 VM passthrough** — DMA attack 가능.

---

## 4. 측정·진단 도구 (cheat sheet)

```bash
# CPU / 메모리 / NUMA
lscpu                              # ISA, 코어, 캐시
lstopo / numactl -H                # 토폴로지
sudo dmidecode -t memory           # DIMM 슬롯
free -h / vmstat 1 / mpstat -P ALL 1

# PCIe / 장치
lspci -tv / lspci -vvv

# 스토리지
lsblk / nvme list
sudo smartctl -a /dev/nvme0n1
fio --rw=randread --bs=4k --iodepth=32 --numjobs=4 --runtime=30 ...

# 네트워크 NIC
ip -s link
ethtool -i / -k / -S / -l eth0
```

---

## 5. 핵심 면접 질문 (역링크)

| 질문 | 답 위치 |
| --- | --- |
| DRAM vs SRAM | [[memory/memory-hierarchy]], [[memory/dram-cell]] |
| HDD vs SSD | [[storage/hdd]], [[storage/ssd-and-ftl]] |
| PCIe lane / gen | [[bus-io/pcie]] |
| DMA / IOMMU | [[bus-io/dma-iommu]] |
| ECC 필요한 경우 | [[memory/ecc-rowhammer]] |
| NAND wear leveling | [[storage/ssd-and-ftl]] |
| Moore's Law 종말 | [[transistor/process-node-evolution]] |
| HBM vs DDR | [[memory/ddr-evolution]] |
| 인터럽트 vs 폴링 | [[bus-io/dma-iommu]] |
| TDP vs 실제 전력 | [[power-cooling/tdp-power]] |
| Rowhammer 가능 이유 | [[memory/ecc-rowhammer]] |
| CXL.mem 중요성 | [[memory/ddr-evolution]] |
| NUMA 영향 | [[soc/soc-anatomy]], 본 hub §3 |
| RAID 5 vs 6 vs 10 | [[storage/raid-levels]] |

---

## 6. 학습 자료 (전체 토픽 공통)

- **Digital Design and Computer Architecture** (Harris & Harris) — 디지털 로직 → 단순 CPU.
- **The Elements of Computing Systems** (Nisan & Schocken) — NAND2Tetris. 무료 강의 동반.
- **Computer Organization and Design** (Patterson & Hennessy) — RISC-V 표준 교재.
- **Computer Architecture: A Quantitative Approach** (Hennessy & Patterson) — 시니어 / 연구.
- **CMOS VLSI Design** (Weste & Harris) — 트랜지스터 회로 단.
- **The Art of Electronics** (Horowitz & Hill) — 회로 일반.
- 강의: **Nand2Tetris** (https://www.nand2tetris.org), MIT 6.004, CMU 15-213 / 18-447, Berkeley CS 152/252.
- 분석 사이트: **WikiChip**, **AnandTech 아카이브**, **Chips and Cheese**, **TechInsights**, **SemiAnalysis**, **Real World Tech**.
- 표준: **JEDEC** (DDR/HBM), **PCI-SIG** (PCIe/CXL), **IEEE 802** (Ethernet/Wi-Fi), **OCP** (데이터센터).

---

## 7. 관련

- [[../computer-architecture/computer-architecture]] — CPU 마이크로아키텍처, 파이프라인, 캐시 일관성, 분기 예측, ISA
- [[../operating-system/operating-system]] — 가상 메모리, 디스크 IO, 스케줄러, 인터럽트 핸들링
- [[../network/network]] — TCP/IP, 라우팅, 스위칭, TLS
- [[../security-theory/security-theory]] — 사이드채널 (Spectre / Meltdown / Rowhammer / Hertzbleed)
- [[../distributed-systems/distributed-systems]] — RDMA, 분산 메모리
- [[../computer-science|↑ computer-science]]
