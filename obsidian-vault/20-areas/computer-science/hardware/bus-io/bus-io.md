---
title: "버스·I/O (Bus / I/O) — Hub"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, bus, io, pcie, usb, thunderbolt, dma, iommu, interrupt]
---

# 버스·I/O — Hub

**[[../hardware|↑ hardware]]**

> CPU·메모리·가속기·NIC·디스크를 잇는 모든 직렬·병렬 인터커넥트.

## 0. 한 줄

**칩과 칩, 칩과 장치, 호스트와 가속기 사이의 모든 신호 통로.** 현대 시스템은 거의 PCIe + 그 위의 확장 (CXL, NVMe) + 외부 (USB, Thunderbolt) + 가속기 전용 (NVLink, Infinity Fabric).

## 1. PCIe — 시스템의 중추
→ [[pcie|PCIe]]

직렬 점대점, lane × generation. PCIe 5.0 x16 = 64 GB/s. CXL 이 그 위에 캐시-일관성 추가.

## 2. USB / Thunderbolt — 외부
→ [[usb-thunderbolt|USB·Thunderbolt]]

USB 1.1 (12 Mbps) → USB4 v2 (120 Gbps). Thunderbolt 가 PCIe x4 터널링.

## 3. DMA / IOMMU / 인터럽트
→ [[dma-iommu|DMA 와 IOMMU]]

CPU 거치지 않는 장치 ↔ 메모리 직접 전송. IOMMU 가 장치의 메모리 접근 권한 격리. MSI/MSI-X 로 인터럽트 vector 다수.

## 4. 가속기 전용 인터커넥트

| 이름 | 출처 | 용도 |
| --- | --- | --- |
| **NVLink 5** (Blackwell) | NVIDIA | 1.8 TB/s per GPU |
| **NVSwitch** | NVIDIA | 8~576 GPU fat-tree |
| **Infinity Fabric 4** | AMD | 코어 / IO / GPU 사이 |
| **UALink** | 컨소시엄 | 2025 NVIDIA 독점 깨려는 표준 |
| **UCIe (Universal Chiplet Interconnect Express)** | 컨소시엄 | 칩렛 사이 die-to-die |

## 5. 칩셋 / PCH

- 옛날: Northbridge (CPU ↔ RAM, ↔ PCIe) + Southbridge (IO).
- 현재: CPU 가 IMC + PCIe + GPU 흡수. **PCH (Intel) / chipset (AMD)** 는 USB / SATA / 추가 PCIe lane 만 제공.
- 데스크탑 보드의 "PCIe Gen5 x4 (M.2)" 가 CPU 직결인지 PCH 경유인지 매뉴얼 확인 — latency 차이.

## 6. SerDes (Serializer / Deserializer)

- 병렬 신호를 직렬화해서 고속 전송 후 다시 병렬화.
- PCIe / USB / Ethernet / HDMI / DisplayPort 모두 SerDes 위에서.
- **PAM4** (1 심볼 2 bit) 가 100G/200G/400G 의 핵심.

## 7. 측정 / 진단

```bash
lspci -tv                       # PCIe 트리
lspci -vvv | grep -E 'LnkCap|LnkSta'   # 협상된 width / speed
lstopo                          # 토폴로지
lsusb -t                        # USB 트리

# DMA / IOMMU
dmesg | grep -i iommu
cat /sys/kernel/iommu_groups/*/devices/*
```

## 8. 함정

1. **lane width 다운그레이드** — PCIe x16 슬롯에 x4 카드 → x4 로만 동작. 또는 x16 슬롯이 BIOS 설정으로 x8/x8 bifurcation.
2. **PCH 경유 슬롯의 DMI 병목** — Intel PCH 는 CPU 와 DMI 4.0 x4 (16 GB/s) 로 연결. 그 아래 모든 USB / SATA / 추가 PCIe lane 가 이 16 GB/s 를 공유.
3. **IOMMU 비활성 채 VM passthrough** — DMA attack. BIOS 에서 VT-d / AMD-Vi 활성.
4. **USB-C 케이블 == Thunderbolt 가정** — 같은 모양이라도 USB 2.0 ~ TB5. 케이블 사양 (5/10/20/40/80/120 Gbps) 확인.
5. **인터럽트 폭주 (NIC)** — 10 GbE 이상은 NAPI / RSS 없으면 한 코어 100% interrupt.

## 9. 관련

- [[../memory/ddr-evolution]] — CXL 의 메모리 풀링
- [[../storage/storage-interfaces]] — NVMe 가 올라타는 layer
- [[../soc/soc]] — chiplet 사이 die-to-die 인터커넥트
- [[../network-hardware/network-hardware]] — NIC PCIe 연결
