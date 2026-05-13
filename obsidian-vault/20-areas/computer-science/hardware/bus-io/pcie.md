---
title: "PCIe (PCI Express)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, bus, pcie, cxl, lane, gt, gbps, bifurcation, serdes, tlp, dllp, aer, aspm, atsdma]
---

# PCIe (PCI Express)

**[[bus-io|↑ 버스·I/O]]**

> 현대 시스템의 중추 인터커넥트. CPU ↔ GPU / NIC / NVMe / 가속기 모두 PCIe 위에. CXL 도 PCIe physical 위에 올라간다.

## 1. 한 줄

**packet 기반 직렬 점대점 인터커넥트.** lane 1 개 = 2 쌍의 차동 신호 (송신 1 쌍 + 수신 1 쌍, full-duplex). PCI 의 병렬 버스 (33 MHz / 66 MHz × 32/64 bit) 를 직렬 + 다중 lane 으로 대체.

## 2. 세대 별 속도 — 표준화된 수치

| 세대 | 출시 | per-lane (GT/s) | 인코딩 | per-lane 유효 (GB/s) | x16 (GB/s) | 변조 |
| --- | --- | --- | --- | --- | --- | --- |
| PCIe 1.0 | 2003 | 2.5 | 8b/10b | 0.250 | 4.0 | NRZ |
| PCIe 2.0 | 2007 | 5.0 | 8b/10b | 0.500 | 8.0 | NRZ |
| PCIe 3.0 | 2010 | 8.0 | 128b/130b | ≈0.985 | 15.75 | NRZ |
| PCIe 4.0 | 2017 | 16 | 128b/130b | ≈1.97 | 31.5 | NRZ |
| PCIe 5.0 | 2019 | 32 | 128b/130b | ≈3.94 | 63.0 | NRZ |
| PCIe 6.0 | 2022 | 64 | FLIT (242B) + PAM4 | ≈7.55 | 121 | PAM4 |
| PCIe 7.0 | 2025 (사양) | 128 | FLIT + PAM4 | ≈15.1 | 242 | PAM4 |

### bandwidth 식 (실 효율)

- **NRZ 시대 (Gen 1–5)**: `유효 GB/s = GT/s × (인코딩 효율) / 8`
  - Gen3 의 128b/130b: `8 × 128/130 / 8 = 0.985`.
- **PAM4 시대 (Gen 6+)**: 1 심볼 2 bit. 같은 baud (32 GBaud) 로 2 배 throughput.
  - 대신 SNR margin 이 NRZ 의 1/3 → FEC (Forward Error Correction) 도입.
  - Gen 6 = FLIT (Flow control unIT, 242 byte) + FEC, 1.3% overhead.

### 양방향 = full-duplex
- 위 표는 단방향. **bidirectional** 은 ×2. 예: PCIe 5.0 x16 bidir = 126 GB/s.
- 마케팅이 한 방향만 표시하는 경우 / 양방향 합산하는 경우 혼재 — 사양서 확인 필수.

## 3. Lane / Width / 협상

| width | 일반 용도 |
| --- | --- |
| x1 | 사운드, 저속 IO, RTC 카드 |
| x4 | NVMe SSD, 일부 NIC |
| x8 | 10/25 GbE NIC, 일부 GPU, capture card |
| x16 | GPU, 데이터센터 NIC (200/400 GbE), NVMe Gen5 enterprise |
| x32 (드물게) | 매우 고성능 (HBA, FPGA 카드) |

### 협상 (Negotiation)

- **down-only 자동 협상**. x16 슬롯에 x4 카드 → x4 로 동작.
- **link speed 협상**: 양쪽 capability 의 최저점.
- 협상 결과 확인:

```bash
sudo lspci -vvv -s 01:00.0 | grep -E 'LnkCap|LnkSta'
# LnkCap:  Port #0, Speed 32GT/s, Width x16   ← 카드/슬롯 max
# LnkSta:  Speed 32GT/s (ok), Width x16 (ok)  ← 실제 협상치
```

### Bifurcation
- BIOS / UEFI 설정으로 x16 슬롯을 `x8 + x8` / `x4 + x4 + x4 + x4` 분리.
- 한 슬롯에 NVMe 4 개 carrier 카드 / 듀얼 GPU 등에 사용.
- 메인보드 + CPU 양쪽 지원 필요. ROOT complex 의 lane 토폴로지가 결정.

### Lane reversal / Polarity inversion
- 보드 routing 최적화를 위해 lane 순서 / 차동 극성을 뒤집어 routing.
- LTSSM (Link Training and Status State Machine) 단계에서 자동 처리.
- 라이저 카드 / 케이블에서 자주 발생.

## 4. 토폴로지 — Root Complex / Switch / Endpoint

```
   CPU (= Root Complex)
   ┌────────┐
   │  IMC   │ ── DRAM channels
   │ Cores  │
   │ Root   ├──── x16 slot 1 (GPU)
   │Complex ├──── x16 slot 2  (또는 x8+x8 분리)
   │        ├──── x4 M.2 (NVMe)
   │        ├──── DMI x4/x8 ──┐
   └────────┘                 │
                              ▼
                        ┌──────────┐
                        │   PCH    │   chipset
                        │          │
                        │ → 추가 PCIe lane (x4 슬롯들)
                        │ → SATA x6
                        │ → USB
                        │ → 추가 M.2
                        └──────────┘
```

- **Root Complex (RC)** — CPU 내부. PCIe fabric 의 root.
- **Switch** — 다운스트림 port 다수. 데이터센터 server motherboard / Thunderbolt 도크.
- **Endpoint (EP)** — leaf 장치 (GPU / NIC / SSD).
- **DMI (Direct Media Interface, Intel)** — CPU ↔ PCH 의 자체 링크. DMI 4.0 x8 = 16 GB/s 양방. **그 아래 PCH 의 모든 USB / SATA / chipset PCIe 가 이 16 GB/s 를 공유**.

### CPU 직결 vs chipset 경유 슬롯

데스크탑 보드의 매뉴얼은 항상 다음 같은 표를 제공:

| 슬롯 | CPU 직결 | chipset 경유 | 비고 |
| --- | --- | --- | --- |
| PCIe x16 _1 | ✓ x16 Gen5 | — | GPU 권장 |
| M.2 _1 | ✓ x4 Gen5 | — | primary NVMe |
| PCIe x16 _2 | ✓ x8 Gen5 (bifurcate) | — | second GPU 시 x16→x8/x8 |
| PCIe x16 _3 | — | x4 Gen4 | chipset 경유, DMI 공유 |
| M.2 _2 ~ _4 | — | x4 Gen4 | chipset, DMI 공유 |

**같은 "x4 Gen4" 라벨이라도 CPU 직결 vs chipset 경유 latency 차이 큼**. 게이밍 / DB / VM IO 가 무거운 워크로드는 직결 슬롯 선택.

## 5. 프로토콜 계층

```
   ┌──────────────────────┐
   │ Software / Driver    │
   ├──────────────────────┤
   │ Transaction Layer    │   TLP (Memory/IO/Config R/W, Message)
   ├──────────────────────┤
   │ Data Link Layer      │   DLLP (ack/nak, FC, retry)
   ├──────────────────────┤
   │ Physical Layer       │   SerDes, scrambling, training, encoding
   └──────────────────────┘
```

### TLP (Transaction Layer Packet)

PCIe 의 "패킷". 5 종류 + 변종:

| TLP Type | 의미 |
| --- | --- |
| **Memory Read / Write** | 가장 흔함. 가상 / 실제 메모리 주소로 R/W. |
| **IO Read / Write** | legacy IO 포트. PCI 호환 용. |
| **Configuration Read / Write** | PCI Config Space 접근. enumerate / capability discovery. |
| **Message** | 인터럽트 (MSI/MSI-X 도 사실 Memory Write TLP), error report, vendor-defined. |
| **Completion** | non-posted (Memory Read, IO, Config R/W) 응답. |

TLP 헤더 = 3 또는 4 DW (12 또는 16 byte). 그 뒤 payload (최대 4 KB, 보통 128~512 byte).

### Non-posted vs Posted

- **Posted**: Memory Write — 응답 없음. 빠름. 순서 보장.
- **Non-posted**: Memory Read / IO / Config — Completion TLP 응답 받아야 끝.

→ 같은 카드에서 read 와 write 가 섞이면 read latency 가 write 보다 훨씬 길다.

### DLLP (Data Link Layer Packet)

- **Ack/Nak** — TLP 도착 확인 / 재전송.
- **FC (Flow Control)** — credit-based. 보내는 쪽이 받는 쪽 buffer 잔량 (credit) 만큼만 전송.
- **Power Management**.

### Credit-based Flow Control

- 송신측은 수신측의 buffer credit 을 추적.
- credit 0 면 송신 대기.
- 수신측이 buffer 비울 때마다 UpdateFC DLLP 로 credit 갱신.
- → drop 없는 lossless 전송.

## 6. Link Training — LTSSM

부팅 시 PCIe link 가 동작 가능한 상태가 되기까지의 상태 머신.

```
Detect      ── 링크 상대 존재 확인
   │
   ▼
Polling     ── 양쪽 신호 교환, 8b/10b sync
   │
   ▼
Configuration ── width / speed 협상
   │
   ▼
L0          ── 활성 데이터 전송 ★ (정상 운영)
   │
   ├── Recovery (속도 변경 / 에러 복구)
   ├── L0s (active idle, ~ns 단위 절전)
   ├── L1  (light sleep, ~μs 진입/복귀)
   ├── L2  (deep sleep, ~ms)
   ├── L3  (off — PCIe reset 또는 power down)
   ▼
Hot Reset / Disabled
```

### ASPM (Active State Power Management)

- L0s / L1 로 자동 전환해 idle 전력 절약.
- L1 은 PHY 까지 꺼짐 → wake latency μs.
- 노트북에서 적극 사용, 데스크탑은 latency 영향 있어 종종 OFF.

## 7. Config Space

각 PCIe 장치는 **PCI Config Space 4 KB** 를 가짐 (PCIe 는 0xFF byte 의 legacy 256B + 0xF00 byte 의 extended).

### 표준 헤더 (Type 0, Endpoint)

| Offset | Field | 의미 |
| --- | --- | --- |
| 0x00 | Vendor ID, Device ID | 카드 식별 |
| 0x04 | Command, Status | enable / 에러 status |
| 0x08 | Class, Subclass, Rev | "Network controller / Ethernet" 같은 카테고리 |
| 0x0C | Cache line, Latency, Header type | |
| 0x10-0x27 | BAR0~BAR5 | Base Address Register (메모리 / IO 영역 위치) |
| 0x2C | Subsystem Vendor / Device | OEM 식별 |
| 0x30 | Expansion ROM base | option ROM (BIOS / UEFI 부팅 코드) |
| 0x34 | Capabilities Pointer | 확장 caps linked list |
| 0x3C | Interrupt Line, Pin | legacy IRQ |

### Capabilities

표준 cap + extended cap 의 linked list. 주요:

| Cap | 의미 |
| --- | --- |
| PCI Express | 본 cap. Link Cap / Status 등 |
| MSI / MSI-X | 인터럽트 표준 |
| Power Management | D0/D1/D2/D3 상태 |
| AER (Advanced Error Reporting) | 상세 에러 보고 |
| Virtual Channels | QoS |
| Single Root I/O Virtualization (SR-IOV) | VF 분할 |
| Access Control Services (ACS) | DMA 격리 (IOMMU 그룹) |
| ATS / PRI / PASID | shared virtual memory |
| Latency Tolerance Reporting | OS 가 ASPM 결정에 활용 |

```bash
sudo lspci -vvv -s 01:00.0    # 전체 config space + capabilities
sudo setpci -s 01:00.0 CAP_EXP+10.W   # specific register 읽기
```

## 8. AER (Advanced Error Reporting)

PCIe 의 에러를 OS / 운영자에게 상세히 보고하는 표준.

### 에러 종류

| 종류 | 예 |
| --- | --- |
| **Correctable** | BER 한 두 비트 — link layer 가 재전송으로 복구. 카운터만 ↑. |
| **Uncorrectable Non-fatal** | TLP 헤더 손상 등 — 한 transaction 만 실패. |
| **Uncorrectable Fatal** | link down, unsupported request — 시스템 안정성 위협. |

### 진단

```bash
# Linux PCIe 에러 카운터
sudo cat /sys/bus/pci/devices/*/aer_dev_*
sudo dmesg | grep -i 'aer\|pcie'

# 자세히 (Mellanox 등)
sudo mlxlink -d /dev/mst/mt4123_pciconf0 --show_eye
```

**CRC 에러 폭증** = 라이저 / 케이블 / 단자 청결 의심. **Gen 다운그레이드** = signal integrity 한계.

## 9. CXL (Compute Express Link) — PCIe 위의 캐시 일관성

PCIe 5.0+ physical layer 위에 3 종 프로토콜.

| 프로토콜 | 의미 | 장치 type |
| --- | --- | --- |
| **CXL.io** | PCIe enumeration + DMA 호환 | 모든 CXL 장치 |
| **CXL.cache** | 가속기가 호스트 메모리 캐시-일관성 R/W | Type 1 (cache only) / Type 2 |
| **CXL.mem** | 호스트가 장치 메모리를 byte-addressable | Type 2 (cache + mem) / Type 3 (mem only) |

### CXL 장치 type

| Type | 가속기 캐시 | 자체 메모리 | 예 |
| --- | --- | --- | --- |
| Type 1 | ✓ | ✗ | SmartNIC, accelerator |
| Type 2 | ✓ | ✓ | GPU / 가속기 |
| Type 3 | ✗ | ✓ | DRAM 익스팬더, persistent memory |

### 메모리 풀링 / 티어링

- **Pooling**: 여러 호스트가 한 CXL.mem 풀 공유.
- **Tiering**: hot 은 DRAM, warm 은 CXL.mem (latency 2-3 배).
- 2025-2026 본격 도입. AI 학습 + DRAM 가격 급등의 대응책.

## 10. UCIe (Universal Chiplet Interconnect Express)

PCIe 가 보드 위 슬롯 사이 / 케이블 사이 라면, **UCIe 는 칩렛 사이 die-to-die**.

- 2022 사양 발표. AMD / Intel / Samsung / TSMC / ARM / Google.
- 같은 패키지 안 칩렛 사이 ns 단위 latency, Tbps 단위 BW.
- PCIe + CXL 의 사상을 패키지 안으로.

## 11. SR-IOV (Single Root I/O Virtualization)

PCIe 장치 1 개를 여러 가상 PCIe 장치로 분할.

```
  Physical NIC (PF, Physical Function)
       │
       ├── VF 1 ──── VM 1 (직접 할당)
       ├── VF 2 ──── VM 2
       ├── VF 3 ──── VM 3
       └── VF n ──── VM n
```

- **PF**: 본체. 드라이버가 관리.
- **VF**: 가상. VM 에 IOMMU 로 격리 후 노출.
- NIC 안 embedded switch 가 VF 사이 트래픽 라우팅.
- 서버 NIC (ConnectX, Intel E810), 일부 GPU (NVIDIA vGPU), NVMe SSD 가 지원.

## 12. ATS / PRI / PASID — Shared Virtual Memory

| Cap | 의미 |
| --- | --- |
| **ATS (Address Translation Services)** | 장치가 IOMMU 의 IOTLB 를 캐시. 가상→실 주소 변환 가속. |
| **PRI (Page Request Interface)** | 장치가 페이지 fault → OS 가 페이지 가져오기. |
| **PASID (Process Address Space ID)** | 같은 장치를 여러 프로세스가 격리해서 공유. |

→ GPU 가 호스트 가상 메모리 공간을 그대로 쓰는 SVM (Shared Virtual Memory) 의 기반.

## 13. 디버깅 — 실 사례 가이드

### 사례 1 — GPU 가 x8 로만 인식
- 의심: 라이저 케이블, 슬롯 lane 공유, 다른 슬롯 카드와 bifurcation, 메인보드 BIOS lane 옵션.
- 확인:
  ```bash
  sudo lspci -vvv -s 01:00.0 | grep -E 'LnkCap|LnkSta'
  ```
  LnkCap 이 x16 인데 LnkSta 가 x8 → 협상 / 케이블 의심.

### 사례 2 — Gen5 SSD 가 Gen4 속도
- 의심: heatsink 부족 → thermal throttling → Gen5 신호 무결성 한계 → Gen4 로 다운그레이드.
- 확인: `smartctl -a` 의 온도, LnkSta speed.

### 사례 3 — AER correctable 카운터 폭증
- 의심: 라이저 / 케이블 / 단자 먼지 / 보드 손상.
- 단기 해법: ASPM disable, link speed 한 단계 ↓.
- 근본 해법: 케이블 교체 / 단자 청소.

### 사례 4 — IOMMU group 강제 분리 (홈랩)
- 같은 group 의 다른 장치가 함께 묶여 VM passthrough 불가.
- `pci=ACS_override=downstream` 부팅 옵션 (보안 trade-off).

## 14. 함정

1. **PCIe Gen5 NVMe 발열** — heatsink 필수. 80 °C 넘으면 throttling.
2. **lane bifurcation 미설정** — 듀얼 NVMe carrier 카드가 한 슬롯만 인식.
3. **CPU 직결 vs PCH 슬롯 혼동** — chipset 경유 슬롯의 DMI 병목 (16 GB/s 공유).
4. **PCIe Gen 다운그레이드** — 1 단 낮으면 throughput 절반. LnkSta 확인.
5. **GPU 가 x8 만 사용 가정** — 게이밍은 그래도 되지만 ML 학습 / HBM ↔ DRAM 트래픽은 x16 필요.
6. **riser cable 손상** — flexible PCIe riser 는 신호 무결성 민감. signal retimer 필요한 경우 있음.
7. **AER non-fatal 누적** — correctable 이 시간당 수천 건 → 곧 fatal 옴. 미리 대처.
8. **MSI/MSI-X 비활성** — legacy IRQ 만 → 인터럽트 storming. capability 확인.
9. **양방향 vs 단방향 bandwidth 혼동** — 마케팅이 합산 표기.
10. **PCIe reset 의도치 않은 발동** — hot reset / FLR (Function Level Reset) 가 NVMe / GPU 의 상태 flush.

## 15. 측정 / 진단 cheatsheet

```bash
# 토폴로지
lspci -tv
lspci -tvnn                              # vendor:device ID 포함

# 자세한 capability + LnkCap/LnkSta
sudo lspci -vvv -s 01:00.0
sudo lspci -vvv | grep -E 'LnkCap|LnkSta|LnkCtl' -A1

# config space dump
sudo lspci -xxx -s 01:00.0 | head -40

# 특정 capability register 읽기
sudo setpci -s 01:00.0 CAP_EXP+10.W       # Link Control

# 에러
sudo dmesg | grep -i 'aer\|pcie'
sudo cat /sys/bus/pci/devices/0000:01:00.0/aer_dev_correctable

# NVMe specific
sudo nvme list
sudo nvme id-ctrl /dev/nvme0n1 | head -30

# 토폴로지 시각화
sudo apt install hwloc
lstopo --of svg > topology.svg
```

## 16. 관련

- [[bus-io]]
- [[dma-iommu]] — MSI-X, IOMMU, SR-IOV 상세
- [[../memory/ddr-evolution]] — CXL.mem
- [[../storage/storage-interfaces]] — NVMe / U.2 / EDSFF / E1.S
- [[../soc/datacenter-accelerators]] — GPU + PCIe + NVLink
- [[usb-thunderbolt]] — Thunderbolt 가 PCIe 터널링
