---
title: "데이터센터 가속기 (H100 / MI300 / TPU / Gaudi)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, accelerator, gpu, h100, h200, blackwell, b100, b200, gb200, mi300, mi325, tpu, gaudi, cerebras, nvlink, nvswitch, hbm, fp8, dgx, hgx]
---

# 데이터센터 가속기

**[[soc|↑ SoC]]**

> 2024-2026 AI 시대의 산업 표준. LLM 학습 / 추론 / HPC 모두 이 가속기들 위에서.

## 1. 가속기 한눈에 (2024-2026)

| 가속기 | 출시 | 공정 | 메모리 | BW (TB/s) | 전력 (W) | FP16 (TFLOPS) | FP8 (TFLOPS) | NVLink/IF (TB/s) |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| **NVIDIA A100** | 2020 | TSMC 7N | 40/80 GB HBM2e | 1.5/2.0 | 400 | 312 (sparse 624) | — | NVLink 3, 0.6 |
| **NVIDIA H100 SXM5** | 2022 | TSMC 4N | 80 GB HBM3 | 3.35 | 700 | 989 (sparse) | 1979 | NVLink 4, 0.9 |
| **NVIDIA H100 NVL** | 2023 | TSMC 4N | 188 GB HBM3 (2 dies) | 7.8 | 700+700 | 1979 | 3958 | NVLink 4 |
| **NVIDIA H200** | 2024 | TSMC 4N | 141 GB HBM3e | 4.8 | 700 | 989 | 1979 | NVLink 4 |
| **NVIDIA B100** | 2024 | TSMC 4NP | 192 GB HBM3e | 8 | 700 | 1750 | 3500 | NVLink 5, 1.8 |
| **NVIDIA B200** | 2024 | TSMC 4NP | 192 GB HBM3e | 8 | 1000 | 2250 | 4500 | NVLink 5, 1.8 |
| **NVIDIA GB200 NVL72** | 2024 | rack | 13.5 TB HBM | 576 | 120 kW | 162000 (FP4 1.4 EF) | — | NVLink 5 + NVSwitch |
| **AMD MI300X** | 2023 | TSMC 5+6 | 192 GB HBM3 | 5.3 | 750 | 1307 | 2614 | Infinity Fabric 4, 0.8 |
| **AMD MI325X** | 2024 | TSMC 5+6 | 256 GB HBM3e | 6 | 1000 | 1307 | 2614 | IF 4, 0.9 |
| **AMD MI350** | 2025 | TSMC 3 | 288 GB HBM3e | 8 | 1000 | 9215 (FP6) | 18432 (FP4) | IF 5 |
| **Google TPU v5p** | 2023 | TSMC | 95 GB HBM | 2.8 | 700 | — | — | ICI |
| **Google TPU v6 (Trillium)** | 2024 | TSMC | 32 GB HBM | 1.6 | — | 4.7× v5e | — | ICI |
| **Intel Gaudi 3** | 2024 | TSMC 5 | 128 GB HBM2e | 3.7 | 900 | 1835 | 1835 | RoCE v2 (NIC built-in) |
| **Cerebras WSE-3** | 2024 | TSMC 5 | 44 GB on-chip SRAM | 21 PB/s on-chip | 23 kW | 125 PFLOPS FP16 | — | WSE-internal |

## 2. NVIDIA Hopper (H100 / H200)

### H100 — 2022 발표, AI 학습 의 게임체인저

| 사양 | 값 |
| --- | --- |
| 트랜지스터 | 80 B |
| Die size | 814 mm² |
| 공정 | TSMC 4N |
| 메모리 | 80 GB HBM3 |
| 메모리 BW | 3.35 TB/s |
| L2 캐시 | 50 MB |
| SM | 132 / 144 |
| FP64 / FP32 / TF32 / FP16 / FP8 | 67 / 67 / 989 / 1979 / 1979 TFLOPS (sparse) |
| TDP | 700 W (SXM5) |
| NVLink | NVLink 4, 900 GB/s/GPU |
| PCIe | Gen5 x16 |

### Transformer Engine
- FP8 (E4M3 / E5M2) 자동 정밀도 관리.
- 동적 range scaling.
- 학습 시 FP16 의 절반 메모리 / 2x throughput.

### Tensor Memory Accelerator (TMA)
- async memory load/store.
- L2 / shared memory ↔ HBM.

### H200 — 2024, 메모리 늘림
- 141 GB HBM3e, 4.8 TB/s.
- compute 는 H100 동일.
- LLM 추론에서 매우 유리 (메모리 bandwidth bound).

## 3. NVIDIA Blackwell (B100 / B200 / GB200)

### Blackwell 의 핵심 변화

1. **듀얼 다이** — B100/B200 은 2 개 die 가 chip-to-chip 10 TB/s 로 결합.
2. **FP4 / FP6** — 새 저정밀도 type. 추론 throughput 2-4× ↑.
3. **HBM3e** — 192 GB / 8 TB/s.
4. **NVLink 5** — 1.8 TB/s per GPU (NVLink 4 의 2 배).
5. **2nd-gen Transformer Engine** — FP4 자동 관리.
6. **Decompression Engine** — Parquet / Snappy / GZIP HW 가속.

### B100 vs B200
- B100 = 700 W. 일부 워크로드 throughput.
- B200 = 1000 W. 풀스펙. 2.25× B100 the H100 의 FP4.

### GB200 — Grace + Blackwell 결합

- **Grace CPU**: ARM Neoverse V2, 72 core, LPDDR5X 480 GB.
- **2 × B200** GPU.
- Grace ↔ B200 = NVLink 900 GB/s (PCIe Gen5 x16 의 14배).
- LPDDR5X 가 GPU 의 swap / extended memory.

### GB200 NVL72 — 72 GPU rack

```
1 rack 안:
- 36 × GB200 Grace+Blackwell 모듈
  = 36 Grace CPU + 72 B200 GPU
- NVLink Switch tray × 9
- 모든 72 GPU 가 직접 fat-tree NVLink (1.8 TB/s/GPU)
- 13.5 TB HBM, 30 TB LPDDR5X
- 120 kW 전력, liquid cooling

LLM 학습 throughput:
- FP4 inference: 1.4 EF (exa-flops, sparse)
- 1.8 trillion-param model 학습 가능
```

## 4. AMD Instinct MI300 / MI325 / MI350

### MI300X — 2023

| 사양 | 값 |
| --- | --- |
| 공정 | TSMC 5N + 6N (chiplet) |
| 트랜지스터 | 153 B |
| 메모리 | 192 GB HBM3 |
| 메모리 BW | 5.3 TB/s |
| FP64 / FP32 / FP16 / FP8 | 163 / 163 / 1307 / 2614 TFLOPS |
| TDP | 750 W |
| Infinity Fabric | 4th gen, 896 GB/s |

### MI300 의 chiplet 구조

```
8 × CCD (GPU compute dies, TSMC 5N)
4 × IOD (I/O die, TSMC 6N)
8 × HBM3 stacks
모두 silicon interposer 위
```

### MI325X — 2024
- HBM3e 256 GB.
- 6 TB/s.
- compute 동일.

### MI350 — 2025
- TSMC 3 nm.
- 288 GB HBM3e.
- FP6 / FP4 추가.
- 9215 / 18432 TFLOPS.

### CDNA vs RDNA
- **CDNA** — datacenter compute GPU (MI 시리즈).
- **RDNA** — gaming GPU (Radeon).
- 다른 아키텍처.

### ROCm / HIP
- AMD 의 CUDA 대응.
- HIP = "CUDA-like" API.
- PyTorch / TensorFlow 가 native 지원 (2023+).
- 단점: 일부 CUDA-specific 라이브러리 호환 부족.

## 5. Google TPU

### TPU v5e (2023, 추론)
- 2x v4 효율.
- 16 GB HBM, 197 TFLOPS BF16.
- 1 pod = 256 chip.

### TPU v5p (2023, 학습)
- 95 GB HBM, 459 TFLOPS BF16.
- 1 pod = 8960 chip.

### TPU v6 (Trillium, 2024)
- 32 GB HBM3.
- 4.7× v5e throughput.
- SparseCore (recommendation 전용).
- 클라우드 (Google Cloud) 전용. 외판 안 함.

### TPU 의 강점
- 큰 systolic array (256×256 또는 더 큼).
- HBM 메모리 BW.
- TPU pod 의 ICI (Inter-Chip Interconnect) 가 NVIDIA NVLink 와 동급.

### TPU 의 약점
- Google Cloud 전용.
- JAX / TF / PyTorch (XLA) 통해서만 사용.

## 6. Intel Gaudi 3 — 2024

| 사양 | 값 |
| --- | --- |
| 공정 | TSMC 5 nm |
| 메모리 | 128 GB HBM2e |
| 메모리 BW | 3.7 TB/s |
| FP16 / FP8 | 1835 / 1835 TFLOPS |
| TDP | 900 W |
| 폼팩터 | OAM |
| 내장 NIC | 24× 200 GbE RoCE v2 |

### 특이점
- 가속기 안에 RDMA NIC 내장 (24 × 200 GbE).
- 외부 NIC 불필요 → all-reduce 가 칩에서 직접.
- 같은 노드 GPU 끼리는 그것으로 통신.

## 7. Cerebras WSE-3 — 단일 wafer-scale

### 사양
- 300 mm wafer 1 장 거의 전체 사용.
- Die size: **46,225 mm²** (단일 GPU 의 60×).
- 트랜지스터 4 trillion.
- 900,000 코어.
- 44 GB on-chip SRAM, 21 PB/s on-chip BW.
- 23 kW.

### 강점
- LLM 학습 시 모델이 한 chip 안에 들어가면 inter-chip 통신 0.
- BW 가 압도적.

### 약점
- 1 칩 = 시스템 1 개. 작은 클러스터.
- 일부 워크로드만 적합.

## 8. NVLink / NVSwitch / Infinity Fabric

### NVIDIA NVLink

| 세대 | per-GPU 양방 | port 수 |
| --- | --- | --- |
| NVLink 1 (P100) | 160 GB/s | 4 |
| NVLink 2 (V100) | 300 GB/s | 6 |
| NVLink 3 (A100) | 600 GB/s | 12 |
| NVLink 4 (H100) | 900 GB/s | 18 |
| NVLink 5 (Blackwell) | 1800 GB/s | 18 |

→ NVLink 5 = PCIe 5.0 x16 (128 GB/s) 의 14 배.

### NVSwitch

- 한 노드 안 GPU fat-tree.
- NVSwitch 1 = V100 시대.
- NVSwitch 2 = A100, 8 GPU.
- NVSwitch 3 = H100, 8 GPU 가 모두 동시 full BW.
- NVSwitch 4 (Blackwell) = rack scale 72 GPU NVL72.

### NVLink Switch Tray (GB200 NVL72)
- 9 개 switch tray = 72 GPU 의 fat-tree.
- 각 GPU 가 동시에 1.8 TB/s 양방.

### AMD Infinity Fabric

- 4th gen = MI300.
- chip 내부 + chip ↔ chip + node ↔ node.
- 1 fabric domain 8 MI300X 가 fully connected.

### UALink — 컨소시엄

- AMD / Intel / Broadcom / Cisco / Google / HPE / Meta / Microsoft (2024).
- NVIDIA NVLink 독점 깨려는 표준.
- Ethernet 위 RDMA + 가속기 coherency.

## 9. 폼팩터

| 폼팩터 | 의미 | 예 |
| --- | --- | --- |
| **PCIe (Gen5 x16)** | 워크스테이션 / 일부 서버 | H100 PCIe, RTX 6000 Ada |
| **SXM (NVIDIA)** | NVLink 직결 모듈 | H100 SXM5, B200 SXM6 |
| **OAM** (Open Accelerator Module) | 표준 모듈 | Gaudi 3, MI300X, MI325X |
| **UBB** (Universal Baseboard) | 8-GPU 호스팅 | HGX 시스템 (8×H100, 8×B200) |
| **Rack-scale (NVL72, MI300X 클러스터)** | 전체 rack 이 한 시스템 | GB200 NVL72, MI300X cluster |

### HGX vs DGX
- **HGX** = NVIDIA 가 OEM 에 제공하는 baseboard. Dell / HPE / Supermicro 등이 chassis + CPU + 네트워크 추가해 완제품 출시.
- **DGX** = NVIDIA 가 직접 만드는 완제품 서버. DGX H100 = 8 H100 + dual Xeon + 30 TB NVMe + 8 NIC.

## 10. 데이터센터 GPU vs 게이밍 GPU

| 비교 | 데이터센터 (H100/B100) | 게이밍 (RTX 4090/5090) |
| --- | --- | --- |
| 메모리 | HBM3e 80~192 GB | GDDR7 24~32 GB |
| FP64 | 가속 (1:2 비) | 거의 없음 (1:64) |
| FP8 / FP4 sparsity | 풍부 | 제한 |
| NVLink | 있음 | 없음 (SLI 단종) |
| ECC | 항상 | 일부만 |
| Tensor Core | 4th/5th gen, sparse | 4th/5th gen |
| 출력 (display) | 없음 (헤드리스) | 4 displays |
| 가격 | $30,000~$50,000 | $1,500~$2,500 |
| 용도 | LLM 학습 / 추론 / HPC | 게임 / 일부 ML 개발 |

### NVIDIA 의 데이터센터 EULA
- GeForce 카드 (RTX 4090 등) 의 데이터센터 사용 금지 (EULA 조항).
- 실제로 단속은 어렵지만 enterprise 라이센스로 H100/A100 강제.

## 11. AI 학습 vs 추론 가속기

| 워크로드 | 가속기 |
| --- | --- |
| 대규모 학습 | H100 / H200 / B100 / B200 / MI300X / MI325 / TPU v5p / Gaudi 3 |
| 대규모 추론 (LLM) | H100 / H200 / B200 / MI300X / Groq LPU / Cerebras / SambaNova |
| 작은 추론 / 엣지 | A10 / L4 / T4 / Jetson / Apple ANE / Qualcomm Hexagon NPU |
| HPC (FP64) | H100 (FP64 67 TF) / MI300X (FP64 163 TF) |

### 추론 전용 가속기
- **Groq LPU** — 단일 칩, deterministic latency.
- **Cerebras CS-3** — WSE-3, 매우 fast inference.
- **SambaNova SN40L** — 1.5 TB DRAM + Reconfigurable Dataflow.

## 12. 학습 워크로드의 메모리 / BW

### 메모리 요구
- 모델 파라미터 (FP16): 7B model = 14 GB.
- 70B model = 140 GB.
- 405B model = 810 GB.
- + gradient (= model size).
- + optimizer state (Adam 의 경우 model size × 2-8).
- + activation (batch / sequence 따라).

→ 70B 모델 학습 = 200-500 GB+ 메모리 → 단일 H100 80 GB 못 올림.

### Parallelism
- **Data parallel** — replicas 가 다른 batch.
- **Tensor parallel** — layer 안 weight 를 GPU 사이 분할.
- **Pipeline parallel** — layer 를 GPU 사이 분할.
- **Sequence parallel** — 시퀀스 길이 차원 분할.

→ 큰 모델 = 3D parallelism (DP + TP + PP) 필수.

### 통신 BW
- AllReduce 가 step time 의 30-70%.
- 같은 노드 안 (NVLink): 900 GB/s ~ 1.8 TB/s.
- 노드 사이 (RoCE v2 / IB): 400-800 GbE = 50-100 GB/s.

→ **노드 사이 BW 가 큰 모델 학습의 천장**.

## 13. DGX H100 / HGX H100 / GB200 — 풀 시스템

### DGX H100 (1 대)
- 8 × H100 SXM5 (80 GB).
- 4-port NVSwitch (fully connected).
- 2 × Intel Xeon Platinum 8480.
- 2 TB DRAM.
- 30 TB NVMe.
- 8 × ConnectX-7 400 GbE.
- 10.2 kW.

### DGX SuperPOD (32 대 = 256 H100)
- 32 × DGX H100.
- 외부 NVLink Switch 로 256 H100 fat-tree.
- 1 EFLOPS FP8.

### GB200 NVL72 (1 rack)
- 36 × GB200 모듈 = 72 GPU + 36 Grace CPU.
- 13.5 TB HBM3e.
- 30 TB LPDDR5X.
- 120 kW liquid cooling.
- rack 한 개 = 1.4 EFLOPS FP4.

## 14. 가격 / TCO

### NVIDIA H100 SXM5 (대략)
- 단가: $30,000~$40,000 (2024 후반).
- 8-GPU HGX 보드: $250,000~$350,000.
- DGX H100 완제품: $400,000+.

### Cloud 시간당
- AWS p5 (8 × H100): $98/시간.
- Azure ND H100 v5: $90/시간.
- GCP A3 Mega (8 × H100): $89/시간.

### 비용 모델
- 학습 1 회 (70B model, 1T token): ~$1M~$5M (cloud).
- 추론 / token: $0.001~0.01.

## 15. 함정

1. **HBM 용량 부족** — 70 B FP16 모델 = 140 GB → 단일 H100 80 GB 못 올림. multi-GPU tensor parallel 필수.
2. **NVLink 없는 PCIe 카드** — GPU 사이 통신 PCIe x16 (64 GB/s) 로 묶임. 학습 throughput 1/14.
3. **데이터센터 GPU 의 fan-less SXM** — 자체 팬 없음. server chassis 의 풍압이 쿨링. 단독 사용 불가.
4. **OAM vs SXM 혼동** — 폼팩터 같은 board 라도 다른 표준.
5. **rack 단위 전원 / 쿨링 미설계** — 120 kW rack 을 일반 데이터센터에 못 들임. liquid + 380/415 V DC.
6. **GeForce 데이터센터 사용** — NVIDIA EULA 위반. enterprise 라이센스 위험.
7. **TPU 외부 사용 시도** — Google Cloud 전용. 자체 클러스터 구축 불가.
8. **AMD MI300X 의 PyTorch 호환** — 일부 CUDA kernel 미호환. ROCm version 확인.
9. **InfiniBand 와 RoCE 혼동** — switch 와 NIC 모드 일치 필수.
10. **GPU thermal throttling** — TJ 90 °C 넘으면 자동 다운. liquid cooling 권장.
11. **CUDA Driver vs CUDA Toolkit 미스매치** — driver 가 최신이면 toolkit 도. nvidia-smi 확인.
12. **NCCL 토폴로지 미검출** — `NCCL_DEBUG=INFO` 로 fabric 사용 확인.

## 16. 진단 / 모니터링

```bash
# NVIDIA
nvidia-smi
nvidia-smi -q -d MEMORY,UTILIZATION,POWER
nvidia-smi topo -m                        # NVLink topology
nvidia-smi nvlink --status

# AMD
rocm-smi
rocm-smi --showtopo
rocm-smi --showtemp

# Intel Gaudi
hl-smi

# CUDA 정보
nvcc --version
cat /usr/local/cuda/version.json

# NCCL 디버깅
NCCL_DEBUG=INFO NCCL_DEBUG_SUBSYS=ALL python train.py

# PyTorch
python -c "import torch; print(torch.cuda.is_available(), torch.cuda.device_count())"
```

## 17. 관련

- [[soc]]
- [[soc-anatomy]]
- [[../memory/ddr-evolution]] — HBM3 / HBM3e / HBM4
- [[../bus-io/pcie]] — PCIe Gen5 + NVLink
- [[../bus-io/dma-iommu]] — SR-IOV, GPU passthrough
- [[../power-cooling/power-cooling]] — rack 단위 전력 / 쿨링
- [[../network-hardware/rdma-infiniband]] — GPU 클러스터 fabric (RoCE / IB)
- [[../../distributed-systems/distributed-systems]] — 분산 학습 / parallelism
