---
title: "PCIe (PCI Express)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, bus, pcie, cxl, lane, gt, gbps, bifurcation, serdes]
---

# PCIe (PCI Express)

**[[bus-io|↑ 버스·I/O]]**

## 1. 한 줄

**packet 기반 직렬 점대점 인터커넥트.** lane 1 개 = 2 쌍의 차동 신호 (송신 1 쌍 + 수신 1 쌍, full-duplex).

## 2. 세대 별 속도

| 세대 | 출시 | per-lane (GT/s) | 인코딩 | per-lane 유효 (GB/s) | x16 (GB/s) |
| --- | --- | --- | --- | --- | --- |
| PCIe 1.0 | 2003 | 2.5 | 8b/10b | 0.25 | 4 |
| PCIe 2.0 | 2007 | 5.0 | 8b/10b | 0.5 | 8 |
| PCIe 3.0 | 2010 | 8.0 | 128b/130b | ≈1.0 | 16 |
| PCIe 4.0 | 2017 | 16 | 128b/130b | ≈2.0 | 32 |
| PCIe 5.0 | 2019 | 32 | 128b/130b | ≈4.0 | 64 |
| PCIe 6.0 | 2022 | 64 (PAM4) | FLIT | ≈8.0 | 128 |
| PCIe 7.0 | 2025 (사양) | 128 (PAM4) | FLIT | ≈16.0 | 256 |

`GT/s (GigaTransfers per second)` × `2 bit/transfer (PAM4 시)` × `lane 수` / `8 bit/byte` ≈ 효율 GB/s.

## 3. lane / width

| width | 일반 용도 |
| --- | --- |
| x1 | 사운드 / 작은 IO 카드 |
| x4 | NVMe SSD, 일부 NIC |
| x8 | 10/25 GbE NIC, 일부 GPU |
| x16 | GPU, 일부 데이터센터 NIC, NVMe Gen5 (서버) |

- 협상은 down-only 양립. x16 슬롯에 x4 카드 → x4 로 동작.
- BIOS / UEFI 의 **bifurcation** 설정으로 x16 슬롯을 x8+x8 또는 x4+x4+x4+x4 분리 가능 (애드인 NVMe carrier 카드, 듀얼 GPU 등).

## 4. PCIe 토폴로지

```
            CPU                       CPU 직결 PCIe
        ┌────────┐
        │  IMC   │ ── DRAM channels
        │ Cores  │
        │  PCIe  ├──── x16 slot 1 (GPU)
        │ Root   ├──── x16 slot 2 (또는 두 슬롯 x8 분리)
        │Complex ├──── x4 M.2 (NVMe)
        │        ├──── DMI x4 ──┐
        └────────┘              │
                                ▼
                          ┌──────────┐
                          │   PCH    │   chipset
                          │          │
                          │  USB     │
                          │  SATA    │
                          │  PCIe x4 │── 추가 슬롯들
                          └──────────┘
```

- **DMI (Direct Media Interface)** = Intel 의 CPU ↔ PCH 링크. DMI 4.0 x8 = 16 GB/s 양방. 그 아래 USB / SATA / 모든 chipset PCIe 가 이 16 GB/s 를 공유.

## 5. PCIe 프로토콜 계층

```
   ┌──────────────────────┐
   │ Software / Driver    │
   ├──────────────────────┤
   │ Transaction Layer    │   TLP (Memory/IO/Config Read/Write)
   ├──────────────────────┤
   │ Data Link Layer      │   ack/nak, retry, flow control
   ├──────────────────────┤
   │ Physical Layer       │   SerDes, scrambling, 8b/10b or 128b/130b
   └──────────────────────┘
```

- **TLP (Transaction Layer Packet)** — Memory Read, Memory Write, IO, Config, Message.
- **MSI / MSI-X** 인터럽트도 TLP (Memory Write to 특별 주소).
- **DLLP (Data Link Layer Packet)** — ack/nak/flow control.

## 6. 협상 / training

부팅 시:
1. **Detect** — 링크 상대 존재 확인.
2. **Polling** — 양 쪽이 신호 교환.
3. **Configuration** — width / speed 협상 (최저 공통점).
4. **L0** — 활성 데이터 전송 상태.

```
lspci -vvv -s 01:00.0 | grep -E 'LnkCap|LnkSta'
# LnkCap:  Speed 32GT/s, Width x16   ← 카드/슬롯 최대치
# LnkSta:  Speed 32GT/s, Width x16   ← 실제 협상치
```

LnkSta 가 LnkCap 보다 낮으면 다음 의심:
- 슬롯이 PCH 경유라 lane 부족.
- 케이블 / riser 손상.
- 다른 슬롯과 lane 공유 (bifurcation 미설정).
- PSU 부족 → 보호 다운그레이드.

## 7. CXL (Compute Express Link)

PCIe 5.0+ 의 physical 위에 캐시-일관성 프로토콜 3 종 (CXL.io / CXL.cache / CXL.mem). → [[../memory/ddr-evolution]] §5.

장치 type:
- **Type 1**: cache only (가속기 / SmartNIC).
- **Type 2**: cache + 자체 메모리 (GPU / 가속기).
- **Type 3**: memory only (DRAM 익스팬더 / persistent memory).

## 8. UCIe (Universal Chiplet Interconnect Express)

칩렛 사이 die-to-die. PCIe 의 사상 (점대점 + 패킷) 을 패키지 내부로. 2022 사양 발표. AMD / Intel / Samsung / TSMC / ARM / Google 컨소시엄.

## 9. 함정

1. **PCIe Gen5 NVMe 발열** — heatsink 없으면 80 °C 도달 → throttling.
2. **lane bifurcation 미설정** — 듀얼 NVMe carrier 카드가 한 슬롯만 인식.
3. **CPU 직결 vs PCH 슬롯 혼동** — PCH 경유 슬롯의 DMI 병목.
4. **PCIe Gen 다운그레이드** — 1 단 낮으면 throughput 절반. LnkSta 확인.
5. **GPU 가 x8 만 사용 가정** — 게이밍은 그래도 되지만 ML 학습 / HBM ↔ DRAM 트래픽은 x16 필요.
6. **riser cable 손상** — flexible PCIe riser 는 신호 무결성 민감. signal retimer 필요한 경우 있음.

## 10. 관련

- [[bus-io]]
- [[dma-iommu]]
- [[../memory/ddr-evolution]] — CXL
- [[../storage/storage-interfaces]] — NVMe / U.2 / EDSFF
- [[../soc/datacenter-accelerators]] — GPU 와 PCIe / NVLink
