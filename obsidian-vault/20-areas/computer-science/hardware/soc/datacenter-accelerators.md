---
title: "데이터센터 가속기 (H100 / MI300 / TPU / Gaudi)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, accelerator, gpu, h100, mi300, tpu, gaudi, cerebras, nvlink, hbm]
---

# 데이터센터 가속기

**[[soc|↑ SoC]]**

## 1. 한눈에

| 가속기 | 출시 | 공정 | 메모리 | 전력 | BW | 비고 |
| --- | --- | --- | --- | --- | --- | --- |
| **NVIDIA A100** | 2020 | TSMC 7N | 40/80 GB HBM2e | 400 W | 1.5–2.0 TB/s | NVLink 3 |
| **NVIDIA H100 SXM5** | 2022 | TSMC 4N | 80 GB HBM3 | 700 W | 3.35 TB/s | NVLink 4 (900 GB/s/GPU) |
| **NVIDIA H200** | 2024 | TSMC 4N | 141 GB HBM3e | 700 W | 4.8 TB/s | NVLink 4 |
| **NVIDIA B100 / B200 (Blackwell)** | 2024-2025 | TSMC 4NP | 192 GB HBM3e | 700 / 1000 W | ~8 TB/s | NVLink 5 (1.8 TB/s/GPU). 듀얼 다이. |
| **NVIDIA GB200 NVL72** | 2024 | rack | 13.5 TB HBM | 120 kW | — | 72 GPU + 36 Grace CPU. |
| **AMD MI300X** | 2023 | TSMC 5/6 | 192 GB HBM3 | 750 W | 5.3 TB/s | Infinity Fabric 7 |
| **AMD MI325X** | 2024 | | 256 GB HBM3e | 1000 W | 6 TB/s | |
| **AMD MI350** | 2025 | | 288 GB HBM3e | 1000 W | — | |
| **Google TPU v5p / v6** | 2023-2025 | — | HBM | — | — | Cloud 전용 |
| **Intel Gaudi 3** | 2024 | TSMC 5 | 128 GB HBM2e | 900 W | — | OAM 폼팩터 |
| **Cerebras WSE-3** | 2024 | TSMC 5 | 44 GB on-chip SRAM | 23 kW | — | 단일 칩 46225 mm² (300 mm wafer 1 장 거의 다 차지) |

## 2. 인터커넥트

### NVIDIA NVLink / NVSwitch

| 세대 | per-GPU 양방 | 비고 |
| --- | --- | --- |
| NVLink 3 (A100) | 600 GB/s | |
| NVLink 4 (H100) | 900 GB/s | |
| NVLink 5 (Blackwell) | 1.8 TB/s | PCIe 5.0 x16 의 ~14 배 |

NVSwitch:
- 8-GPU 노드 안 fat-tree.
- GB200 NVL72: 72 GPU 를 한 rack 안 NVSwitch fabric 으로.

### AMD Infinity Fabric

- 코어 ↔ IO ↔ 메모리 ↔ 다른 가속기.
- MI300X 8-GPU 노드는 8x6 fully-connected.

### UALink / Ultra Ethernet

- 2025 컨소시엄. NVIDIA NVLink 독점 깨려는 표준.
- Ethernet 위 RDMA + 가속기 코히어런시.

## 3. 폼팩터

| 폼팩터 | 의미 | 예 |
| --- | --- | --- |
| **PCIe (Gen5 x16)** | 워크스테이션 / 일부 서버 | H100 PCIe |
| **SXM** (NVIDIA) | NVLink 직결 모듈 | H100 SXM5 |
| **OAM (Open Accelerator Module)** | 표준 모듈 | Gaudi 3, MI300X |
| **UBB (Universal Baseboard)** | 8-GPU 호스팅 | HGX 시스템 |
| **Rack-scale (NVL72 등)** | 전체 rack 이 한 시스템 | GB200 NVL72 |

## 4. 전력 / 쿨링

- 단일 가속기 700–1000 W.
- DGX H100 = 10.2 kW (8 GPU + CPU + NIC).
- GB200 NVL72 = 120 kW per rack.
- 풍랭 한계 초과 → **DLC (Direct Liquid Cooling)** 또는 **immersion** 으로 전환.
- 한 rack 의 전력·쿨링이 데이터센터 floorplan 을 결정.

## 5. 데이터센터 GPU vs 게이밍 GPU

| 비교 | 데이터센터 (H100/B100) | 게이밍 (RTX 4090/5090) |
| --- | --- | --- |
| 메모리 | HBM3e 80~192 GB | GDDR7 24~32 GB |
| FP64 | 가속 | 거의 없음 |
| FP8 / FP16 sparsity | 풍부 | 제한적 |
| NVLink | 있음 | 없음 / SLI 단종 |
| ECC | 항상 | 일부만 |
| 가격 | 30,000~50,000 USD | 2,000~3,000 USD |
| 용도 | LLM 학습 / 추론 / HPC | 게임 / 일부 ML 개발 |

## 6. 추론 vs 학습 가속기

| 워크로드 | 가속기 |
| --- | --- |
| 학습 | H100 / H200 / Blackwell / MI300X / TPU v5p |
| 대규모 추론 | H100 / H200 / Groq LPU / Cerebras / SambaNova |
| 모바일 / 엣지 추론 | Apple ANE / Qualcomm Hexagon NPU / Intel NPU |

## 7. 함정

1. **HBM 용량 부족** — 70 B 파라미터 FP16 모델 = 140 GB → 단일 H100 80 GB 못 올림. 4 GPU + tensor parallel 필요.
2. **NVLink 없는 PCIe 카드** — GPU 사이 통신이 PCIe x16 64 GB/s 로 묶임. 학습 throughput 저하.
3. **데이터센터 GPU 의 fan-less SXM** — 자체 팬 없음. 서버 chassis 의 풍압이 쿨링.
4. **OAM vs SXM 혼동** — 폼팩터 같은 board 라도 다른 표준.
5. **rack 단위 전원 / 쿨링 미설계** — 120 kW rack 을 일반 데이터센터에 못 들임. liquid + 380 V/415 V DC 전원 필수.

## 8. 관련

- [[soc]]
- [[soc-anatomy]]
- [[../memory/ddr-evolution]] — HBM
- [[../bus-io/pcie]] — PCIe / NVLink / NVSwitch
- [[../power-cooling/power-cooling]] — rack 전력 / 쿨링
