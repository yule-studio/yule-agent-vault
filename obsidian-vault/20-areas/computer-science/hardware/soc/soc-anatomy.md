---
title: "SoC 구성 요소"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, soc, anatomy, npu, dsp, isp, codec, modem, fabric, noc, numa]
---

# SoC 구성 요소

**[[soc|↑ SoC]]**

```
┌────────────────────────────────────────────────────┐
│ SoC                                                │
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
│  │  Memory Controller / Cache (SLC) / Fabric  │    │
│  └────────────────────────────────────────────┘    │
│                                                    │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐               │
│  │ USB  │ │ PCIe │ │Display│ │ I/O  │              │
│  └──────┘ └──────┘ └──────┘ └──────┘               │
└────────────────────────────────────────────────────┘
```

## 1. CPU 영역

- 모바일·노트북 SoC = **big.LITTLE** (Performance + Efficiency core).
- Apple M4: 4 P + 6 E. 부하 따라 스케줄러가 분배.
- ARM ISA 거의 표준, x86 (Intel/AMD), RISC-V 일부 임베디드.

## 2. GPU

- 통합 GPU 가 화면 + 일부 GPGPU.
- Apple GPU: Tile-Based Deferred Rendering.
- 모바일 ARM Mali / Qualcomm Adreno / Imagination PowerVR.

## 3. NPU (Neural Processing Unit)

- INT8 / FP16 / INT4 matmul 가속.
- 모바일: 30~50 TOPS (2024).
- AMD XDNA2 NPU (Strix Point) ~50 TOPS.
- Intel NPU (Meteor Lake) ~11, Lunar Lake ~48.

## 4. DSP (Digital Signal Processor)

- 음성 인식 (always-on listening), 노이즈 캔슬링, sensor fusion.
- 작은 면적·낮은 전력으로 background 동작.

## 5. ISP (Image Signal Processor)

- 카메라 센서의 raw 데이터 → 노출 / WB / 디노이즈 / 색 / HDR 합성 / 얼굴 인식.
- 모바일 SoC 의 사진 품질 차이의 핵심.
- Apple ISP, Qualcomm Spectra, Google IPU, Pixel Visual Core.

## 6. Codec HW

- H.264 / H.265 / AV1 / VP9 / JPEG / WebP 인코딩·디코딩 전용 블록.
- 소프트웨어 디코딩 대비 100× 낮은 전력.

## 7. Modem

- 4G LTE / 5G NR / Wi-Fi / Bluetooth.
- 모바일: SoC 안 또는 별도 칩 (Apple A18: external Qualcomm modem 또는 자체 C1 modem).
- 노트북: M.2 WWAN 카드 또는 SoC 통합.

## 8. Secure Enclave / TPM-equivalent

- 키 / 생체 인식 / 결제 토큰 격리.
- Apple Secure Enclave, Android TEE (TrustZone), Intel SGX/TDX, AMD SEV.

## 9. Fabric / NoC (Network on Chip)

- 위 블록들을 잇는 내부 네트워크.
- 라우터 + 링크 + QoS 정책.
- AMD Infinity Fabric, Intel Mesh / Ring, Apple custom.

## 10. 메모리 컨트롤러 / 캐시

- 옛 Northbridge 의 IMC 가 CPU 안으로.
- L3 위 **System Level Cache (SLC)** 가 CPU / GPU / NPU 공유 (Apple 16~48 MB).
- 모바일 SoC: LPDDR5/X 패키지 내부 (PoP / SiP).

## 11. NUMA on SoC

- 멀티 다이 / 멀티 소켓 SoC 는 NUMA.
- 한 다이의 메모리가 빠름. 다른 다이는 fabric 경유.
- Apple M2/M3/M4 Ultra = 두 다이 (UltraFusion ≈ 2.5 TB/s).
- AMD EPYC = N CCD + IOD. 같은 CCD 안 코어끼리 캐시 공유, 다른 CCD 와는 IOD 경유.

```bash
numactl -H              # NUMA 토폴로지
lstopo                  # 시각화
```

## 12. 함정

1. **NUMA 무시한 컨테이너 / VM 배치** — cross-die 메모리 접근으로 latency 폭증.
2. **NPU TOPS 마케팅** — 정밀도 (INT4/INT8/FP16) 와 sparsity 가정에 따라 숫자 다름.
3. **모바일 SoC 의 sustained 성능 ≠ peak** — 첫 1-2 분 burst 후 thermal throttling.
4. **iGPU 가 dGPU 대체** — VRAM BW 가 1/10 이라 게임 / ML 워크로드에서 다름.

## 13. 관련

- [[soc]]
- [[datacenter-accelerators]]
- [[../memory/ddr-evolution]] — LPDDR / HBM / SLC
- [[../../computer-architecture/computer-architecture]] — CPU 마이크로아키텍처
