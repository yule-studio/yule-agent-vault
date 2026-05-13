---
title: "네트워크 하드웨어 (Network Hardware) — Hub"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, network, nic, dpu, smartnic, switch, rdma, infiniband, ethernet]
---

# 네트워크 하드웨어 — Hub

**[[../hardware|↑ hardware]]**

> 프로토콜 / OSI / IP / TCP / TLS / HTTP 등 소프트웨어 레이어는 [[../../network/network]].
> 이 hub 는 NIC, 스위치, 트랜시버, RDMA, DPU 같은 **하드웨어 면**.

## 0. 한 줄

**호스트와 외부 세계 사이의 광·동·무선 통신을 담당하는 모든 칩과 카드.**

## 1. NIC 가속 기능
→ [[nic-offload|NIC 가속 기능]]

RSS / TSO / GRO / LRO / GSO / Checksum / TLS offload / RDMA / DPDK.

## 2. DPU / SmartNIC

- NIC + ARM CPU + 가속기.
- 가상화 / 보안 / 스토리지 처리를 호스트 CPU 에서 떼어냄.
- **NVIDIA BlueField-3** — 22 ARM core + 400 GbE + crypto.
- **AWS Nitro** — DPU 로 EC2 호스트 OS 를 거의 비움.

## 3. 스위치 / 패브릭
→ [[switch-fabric|스위치·패브릭]]

L2 스위치 (MAC), L3 스위치 / 라우터 (IP). 데이터센터: Spine-Leaf / Fat-tree / Dragonfly. ASIC 단일 칩 51.2 Tbps (Broadcom Tomahawk 5).

## 4. RDMA / InfiniBand
→ [[rdma-infiniband|RDMA·InfiniBand]]

원격 호스트 메모리 ↔ 로컬 메모리 직접. RoCE v2 (Ethernet) / iWARP / native InfiniBand.

## 5. 광 / SerDes / 트랜시버

| 폼팩터 | 속도 | 비고 |
| --- | --- | --- |
| SFP / SFP+ | 1 / 10 GbE | 일반 |
| SFP28 | 25 GbE | |
| QSFP+ | 40 GbE | 4×10 |
| QSFP28 | 100 GbE | 4×25 |
| QSFP-DD | 200/400 GbE | 8×25 / 8×50 |
| OSFP | 400/800 GbE | 8×50 / 8×100 |
| **CPO (Co-Packaged Optics)** | 800 GbE+ | 광 트랜시버를 ASIC 옆 패키지 통합 |
| **DAC (Direct Attached Copper)** | 10–400 GbE | 짧은 거리 (3-5 m) 동선 |
| **AOC (Active Optical Cable)** | 10–400 GbE | 광 + 전기 일체형 케이블 |

**PAM4** = 1 심볼 2 bit. 50G/100G/200G per lane.

## 6. 무선

| 표준 | 대역 | 속도 |
| --- | --- | --- |
| Wi-Fi 6 (802.11ax) | 2.4/5 | 9.6 Gbps 이론 |
| Wi-Fi 6E | 2.4/5/6 | 6 GHz 추가 |
| Wi-Fi 7 (802.11be) | 2.4/5/6 | 46 Gbps 이론. MLO. 4096-QAM |
| Bluetooth 5.4 | 2.4 | LE Audio, Auracast |
| UWB (Apple U1/U2) | 6.5–8 GHz | cm 단위 거리·방향 |
| 5G NR | sub-6 / mmWave | up to 10 Gbps |

## 7. 함정

1. **케이블이 보틀넥** — CAT 5e 로 10 GbE 불가. CAT 6A 필요.
2. **NIC 가 PCIe lane 부족** — 100 GbE NIC 는 PCIe 4.0 x16 권장.
3. **MTU 미스매치** — jumbo frame 9000 가 일부 hop 에서 1500 으로 fragment → throughput 폭락.
4. **RSS / RPS 없이 단일 코어 처리** — 10 GbE+ 에서 한 코어 100%.
5. **RoCE 의 PFC / ECN 미설정** — packet drop 으로 throughput ↓.
6. **rack 안 fiber 청결도 무시** — 광 단자 먼지로 BER ↑.

## 8. 관련

- [[nic-offload]]
- [[switch-fabric]]
- [[rdma-infiniband]]
- [[../bus-io/pcie]] — NIC 가 올라타는 layer
- [[../bus-io/dma-iommu]] — RSS / NAPI / SR-IOV
- [[../../network/network]] — 프로토콜 레이어
