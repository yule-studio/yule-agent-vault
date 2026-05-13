---
title: "SoC 구성 요소"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, soc, anatomy, cpu, gpu, npu, dsp, isp, codec, modem, fabric, noc, numa, apple-silicon, uma]
---

# SoC 구성 요소

**[[soc|↑ SoC]]**

> 한 칩 안에 컴퓨터 한 대 — CPU·GPU·NPU·DSP·ISP·Modem·메모리 컨트롤러·I/O 까지 통합.

## 1. SoC 전체 구조

```
┌────────────────────────────────────────────────────┐
│ SoC Package                                        │
│                                                    │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐               │
│  │ CPU  │ │ GPU  │ │ NPU  │ │ DSP  │               │
│  │P + E │ │      │ │      │ │      │               │
│  └──────┘ └──────┘ └──────┘ └──────┘               │
│                                                    │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐               │
│  │ ISP  │ │Codec │ │Modem │ │Secure│               │
│  │      │ │ HW   │ │ 5G   │ │Enclv │               │
│  └──────┘ └──────┘ └──────┘ └──────┘               │
│                                                    │
│  ┌────────────────────────────────────────────┐    │
│  │  Memory Controller / SLC / Fabric (NoC)    │    │
│  └────────────────────────────────────────────┘    │
│                                                    │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐               │
│  │ USB  │ │ PCIe │ │Display│ │ I/O  │              │
│  └──────┘ └──────┘ └──────┘ └──────┘               │
│                                                    │
│  ┌────────────────────────────────────────────┐    │
│  │ LPDDR5X (패키지 내부 또는 PoP)             │    │
│  └────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────┘
```

## 2. CPU 영역

### big.LITTLE 아키텍처 (ARM big.LITTLE / DynamIQ)

- **Performance core (P-core)** — 큰 OOO core, 고성능, 고전력.
- **Efficiency core (E-core)** — 작은 인-order, 저전력.
- 스케줄러가 부하 따라 자동 분배.

| 칩 | P-core 수 | E-core 수 |
| --- | --- | --- |
| Apple A18 Pro | 2 | 4 |
| Apple M4 | 4 | 6 |
| Apple M4 Pro | 10 | 4 |
| Apple M4 Max | 12 | 4 |
| Snapdragon 8 Elite | 2 (Phoenix L) | 6 (Phoenix M) |
| Snapdragon X Elite (laptop) | 12 (Oryon) | — |
| Intel Lunar Lake (laptop) | 4 (P) | 4 (E) |
| Intel Arrow Lake (desktop) | 8 (P) | 16 (E) |

### Apple P-core 의 특징

- 매우 wide OOO (8-decode, 600+ ROB entry).
- branch predictor 의 강함이 단일 코어 성능의 핵심.
- L1 = 128 KB+ (Intel/AMD 의 ~2 배).
- 효율적 ISA (ARM64 의 고정 4-byte instruction) 가 decode 단순화.

### Intel Performance + Efficiency 의 미묘함

- AVX-512 / AMX 가 일부 E-core 미지원 → 같은 ISA 안에서 capability 차이.
- Intel Thread Director — OS 가 스케줄링 힌트 받음.

## 3. GPU

### 모바일 SoC 의 GPU

| 종류 | 출처 | 특징 |
| --- | --- | --- |
| **Apple GPU** | Apple 자체 | Tile-Based Deferred Rendering (TBDR). 메모리 BW 효율 ↑. |
| **ARM Mali** | ARM | 다양한 series (G7xx, G8xx). 라이센서. |
| **Qualcomm Adreno** | Qualcomm | 8 Elite 의 Adreno 830. |
| **Imagination PowerVR** | Imagination | MediaTek 일부 사용. |
| **AMD RDNA** (Samsung Exynos) | AMD 라이센스 | Exynos 2400 의 Xclipse 940. |

### GPU 성능 지표

- **TFLOPS FP32** — 비교 가능 베이스.
- **메모리 BW** — 종종 TFLOPS 보다 더 결정적.
- **삼각형 / sec** — 그래픽 워크로드.

Apple M4 Pro GPU: ~9 TFLOPS FP32, 273 GB/s LPDDR5X.

### GPU 의 unified shader
- vertex / pixel / compute shader 모두 같은 SIMD 유닛.

### Ray Tracing HW
- Apple A17 Pro / M3+ — HW RT.
- Qualcomm Adreno 740+ — HW RT.
- 모바일 게임 / pro app 의 RT 가속.

## 4. NPU (Neural Processing Unit)

INT8 / FP16 / INT4 matmul 가속.

| 칩 | NPU 이름 | TOPS (INT8) |
| --- | --- | --- |
| Apple A18 Pro | Neural Engine 16-core | 35 |
| Apple M4 | Neural Engine 16-core | 38 |
| Snapdragon 8 Elite | Hexagon NPU | 45 |
| Snapdragon X Elite | Hexagon NPU (Copilot+) | 45 |
| Intel Lunar Lake | NPU 4 | 48 |
| AMD Strix Point | XDNA 2 NPU | 50 |
| Google Tensor G4 | TPU 4th gen | — |
| MediaTek Dimensity 9400 | APU 890 | 50 |

### Microsoft Copilot+ PC 요건
- NPU TOPS ≥ 40 (INT8).
- 16 GB RAM.
- 256 GB SSD.
- 첫 적용: Snapdragon X Elite, Lunar Lake, Strix Point.

### NPU 구조 (Apple Neural Engine 추정)

- 16 core × 8-bit × 256 MAC = 4096 MAC.
- 1.5 GHz × 2 ops (MAC) × 4096 × 16 ≈ 200 TFLOPS INT8 (theoretical).
- 실제 사용 TOPS 는 좀 낮음 (35-38).

### NPU 의 사용처
- iPhone / Mac 의 사진 분석.
- Voice (Siri).
- Live Text (OCR).
- 자연어 (deep search).
- Whisper / Llama on-device.

## 5. DSP (Digital Signal Processor)

- 항시 켜진 background DSP.
- 음성 인식 ("Hey Siri" / "Hey Google").
- 노이즈 캔슬링.
- sensor fusion (acelero + gyro + mag + GPS).
- 매우 낮은 전력 (μW 단위).

## 6. ISP (Image Signal Processor)

카메라 raw → 화면 픽셀.

### 처리 단계
1. **demosaic** — Bayer pattern (RGGB) → RGB.
2. **noise reduction**.
3. **lens correction** (왜곡, vignette).
4. **HDR tone mapping**.
5. **face / scene detection**.
6. **encoding** (JPEG / HEIF / ProRAW).

### 모바일 SoC 의 ISP 가 사진 차이의 핵심

- Apple ISP — Smart HDR, Photonic Engine.
- Qualcomm Spectra — Snapdragon 의 ISP.
- Google IPU / Pixel Visual Core — Pixel 의 Night Sight.
- 같은 sensor 라도 ISP 가 결과 좌우.

### computational photography
- 한 노출 = 1 photo 가 아닌, 여러 burst frame 을 합성.
- Apple Deep Fusion, Google Night Sight, Samsung Adaptive Pixel.
- 1 사진 = NPU + ISP + CPU 의 협력.

## 7. Video Codec HW

- H.264 / H.265 (HEVC) / AV1 / VP9 / JPEG / WebP 전용 인코더·디코더.
- **소프트웨어 디코딩 대비 100× 낮은 전력**.
- 1 video frame = 1 mJ vs 100 mJ.

### Apple Media Engine (M3+)
- ProRes / ProRes RAW 가속.
- 비디오 편집 워크로드의 핵심.

### AV1 가속
- 2024+ 모바일 SoC 가 AV1 decode 표준.
- YouTube / Netflix 4K AV1 streaming 가능.
- Apple M3+, Snapdragon 8 Gen 2+, MediaTek Dimensity 9000+.

## 8. Modem — 4G / 5G

| 칩 | Modem | 출시 |
| --- | --- | --- |
| Apple A18 Pro | Qualcomm Snapdragon X75 (external) | 2024 |
| Apple A19 (예정) | Apple C1 modem (자체) | 2025 |
| Qualcomm Snapdragon | Snapdragon X80 integrated | 2024 |
| Samsung Exynos 2400 | Samsung Exynos Modem 5400 | 2024 |
| Google Tensor G4 | Samsung Modem 5300 | 2024 |
| MediaTek Dimensity 9400 | MediaTek M85 | 2024 |

### Apple 의 modem 자체 개발 (C1)
- 2019 Intel modem 사업부 인수.
- 5 년 개발 후 2025 iPhone 16e 에 첫 적용 (C1).
- Qualcomm 의존도 줄임.

## 9. Secure Enclave / TEE

키 / 생체 / 결제 토큰 격리.

| 시스템 | 이름 |
| --- | --- |
| Apple | Secure Enclave |
| Android | TEE / Trusted Execution Environment (TrustZone) |
| Intel | SGX (Software Guard Extensions), TDX (Trust Domain Extensions) |
| AMD | SEV (Secure Encrypted Virtualization), SEV-SNP |
| ARM | TrustZone, CCA (Confidential Compute Architecture) |

### Apple Secure Enclave (SEP)
- 별도 코어 + 별도 메모리.
- 결제 (Apple Pay), 생체 (Face ID), 키체인 보호.
- 외부 메모리 접근 불가.
- 자체 OS (sepOS).

### Confidential Compute
- 2024+ 트렌드. cloud 의 VM 이 hypervisor 도 못 보는 상태로 실행.
- Intel TDX, AMD SEV-SNP, Apple Private Cloud Compute.

## 10. Fabric / NoC (Network on Chip)

위 블록들을 잇는 내부 네트워크.

### 구성 요소
- **Router** — 각 IP block 에 attached.
- **Link** — router 사이.
- **QoS policy** — high-priority traffic 우선.
- **Coherency** — 캐시 일관성 (MESI / MOESI).

### 대표
- **AMD Infinity Fabric** — 코어 ↔ IO ↔ 메모리 ↔ 가속기.
- **Intel Mesh** / **Ring** — Xeon 의 ring (옛) / 2D mesh (Skylake+).
- **Apple custom NoC** — fabric 위 unified memory.

### 진화
- ring → 2D mesh → fat-tree → wafer-scale fabric (Cerebras WSE).

## 11. 메모리 컨트롤러 — IMC

옛 Northbridge 의 IMC 가 CPU 안으로 통합.

### IMC 진화
- Intel: Nehalem (2008) 부터.
- AMD: Athlon 64 (2003) 부터.
- Apple Silicon: 처음부터.

### IMC 의 역할
1. DDR/LPDDR 명령 발행.
2. Refresh 자동 처리.
3. ECC 계산 / 검증 (지원 시).
4. Bank scheduling.
5. NUMA 의 한 부분.

## 12. SLC (System Level Cache)

L3 위, IMC 와 fabric 사이의 큰 공유 캐시.

### Apple Silicon 의 SLC
- M1 Pro: 24 MB.
- M1 Max: 48 MB.
- M2 / M3 / M4: 비슷한 크기.
- CPU / GPU / NPU 모두 공유.

### 효용
- GPU 의 작은 working set 이 SLC 안에 머무름.
- GPU ↔ CPU 데이터 sharing 시 fabric 없이 SLC 에서.

### Intel 의 LLC
- L3 가 SLC 역할.
- CPU 만 사용 (iGPU 는 일부 access).

### AMD Ryzen 의 V-Cache
- L3 위 추가 적층 (TSV).
- 7950X3D = 96 MB L3 (32 MB + 64 MB V-cache).
- 게이밍 / DB 의 working set 이 L3 안에 머무름.

## 13. UMA (Unified Memory) — Apple Silicon 의 특이점

```
       LPDDR5X (메모리 풀)
            ▲
            │ 직접 access
   ┌────────┼────────┐
   │        │        │
  CPU      GPU      NPU
   │        │        │
   └────────┴────────┘
        같은 메모리, 같은 주소 공간
```

### 효과
- CPU / GPU / NPU 가 같은 메모리.
- DRAM ↔ GPU 사이 copy 없음 (PC 의 dGPU 와 비교).
- ML 추론 / 비디오 편집 / 가상 카메라 같은 mixed 워크로드 효율.

### 한계
- 메모리 BW 가 유한 — 모두가 동시 활용 못 함.
- 단일 칩 메모리 용량 한계 (M4 Max 128 GB).

## 14. NUMA + SoC

대형 SoC = 칩렛 / 다수 다이 → NUMA.

### Apple M Ultra
- M Max 두 다이를 **UltraFusion** 으로 결합.
- 다이 사이 BW 2.5 TB/s.
- 같은 다이 메모리 = 빠름.
- 다른 다이 메모리 = 약간 느림 (~1.5x).

### AMD Ryzen / EPYC
- N × CCD (Core Complex Die) + 1 IOD.
- 같은 CCD 안 코어 = L3 공유.
- 다른 CCD 와 = IOD 경유 → 더 긴 latency.

### Intel Xeon / Sapphire Rapids
- 4 tile 구성 (multi-die).
- mesh 가 tile 가로질러 확장.
- 같은 tile = 빠름.

### NUMA 측정

```bash
numactl -H
lstopo --of console
sudo apt install numastat
numastat -m
```

## 15. SoC 의 OS-level 인식

### Linux 의 SoC topology
- `/sys/devices/system/node/` — NUMA 노드.
- `/sys/devices/system/cpu/cpu*/topology/` — 코어 / 패키지 / 스레드.
- `/sys/kernel/iommu_groups/` — IOMMU.

### macOS
- `sysctl -n hw.perflevel0.physicalcpu` — P-core 수.
- `sysctl -n hw.perflevel1.physicalcpu` — E-core 수.
- `sysctl -n hw.l1icachesize` — L1 instruction cache.

## 16. 함정

1. **UMA = 모든 가속기 동시 사용** 오해 — 같은 DRAM 풀이지만 BW 는 유한.
2. **NPU TOPS 직접 비교** — INT8 vs FP16 vs INT4 의 TOPS 가 모두 다른 정밀도. sparsity 가정도 다름.
3. **NUMA 무시한 컨테이너 / VM 배치** — cross-die 메모리로 latency 폭증.
4. **Apple Silicon = 영원히 호환** 가정 — 같은 OS 라도 메이저 업그레이드마다 ABI 미세 변경.
5. **iGPU 가 dGPU 대체** — VRAM BW 가 1/10. ML 학습 못 함.
6. **모바일 SoC sustained 성능** — 첫 1-2 분 burst 후 thermal throttling. burst 만 측정한 벤치 신뢰 ↓.
7. **Modem 통합 = 항상 더 빠르다** 가정 — 일부 워크로드 (5G) 는 external modem 이 더 안정.
8. **Secure Enclave 우회 가능** 오해 — 매우 어렵다. 단, side channel (Apple GoFetch 2024) 같은 새 공격 발견.
9. **DSP / ISP 의 power 측정** — always-on 이라 sleep mode 의 큰 부분. battery life 분석 시 고려.
10. **P-core 와 E-core 간 ISA 차이** — Intel 의 AVX-512 / AMX 가 E-core 미지원 → 응용 fallback 필요.

## 17. 진단

```bash
# Linux
lscpu                                # CPU topology
lstopo --of svg > topo.svg           # 시각화
numactl -H

# Apple Silicon
sysctl hw                            # CPU / cache
system_profiler SPHardwareDataType   # 전체
powermetrics --samplers smc          # 전력
powermetrics --samplers cpu_power    # CPU 활용

# Android
adb shell cat /proc/cpuinfo
adb shell cat /sys/devices/system/cpu/cpufreq/policy0/scaling_cur_freq
```

## 18. 관련

- [[soc]]
- [[datacenter-accelerators]] — 데이터센터 GPU / 가속기
- [[../memory/ddr-evolution]] — LPDDR / HBM / UMA / Apple Silicon BW
- [[../memory/memory-hierarchy]] — SLC / L3 / SLC sharing
- [[../bus-io/pcie]] — PCIe / NVLink / die-to-die
- [[../../computer-architecture/computer-architecture]] — CPU 마이크로아키텍처
