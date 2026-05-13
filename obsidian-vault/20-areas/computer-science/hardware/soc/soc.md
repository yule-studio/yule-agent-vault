---
title: "SoC (System on Chip) — Hub"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, soc, npu, dsp, apple-silicon, accelerator, nvlink]
---

# SoC — Hub

**[[../hardware|↑ hardware]]**

> CPU + 주변 블록을 한 다이 / 한 패키지로. 모바일·노트북·서버·자동차·임베디드 거의 모든 현대 시스템.

## 0. 한 줄

**CPU·GPU·NPU·DSP·ISP·Codec·Modem·Memory controller·I/O 가 한 칩에 묶인 통합 칩.**

## 1. SoC 구성 요소
→ [[soc-anatomy|SoC 구성 요소]]

CPU 코어 (P + E), GPU, NPU, DSP, ISP, codec HW, modem, secure enclave, fabric (NoC), 메모리 컨트롤러.

## 2. 모바일 SoC (2024-2026)

| 칩 | 공정 | 코어 | NPU TOPS |
| --- | --- | --- | --- |
| Apple A18 Pro | TSMC N3E | 6 (2P+4E) | 35 |
| Apple M4 | TSMC N3E | 10 (4P+6E) | 38 |
| Snapdragon 8 Elite (2024) | TSMC N3E | Oryon 8 | 45 |
| Snapdragon X Elite (Windows) | TSMC N4 | Oryon 12 | 45 |
| Google Tensor G4 | Samsung 4LPP+ | 8 | — |
| MediaTek Dimensity 9400 | TSMC N3E | 1+3+4 | 50 |

## 3. Apple Silicon

- **UMA (Unified Memory)** — CPU/GPU/NPU 같은 메모리 풀.
- **시스템 레벨 캐시 (SLC)** 16~48 MB 공유.
- **AMX** — CPU 옆 행렬 가속기.
- M Ultra = M Max 2 개 UltraFusion 으로 결합.

## 4. 데이터센터 가속기
→ [[datacenter-accelerators|데이터센터 가속기]]

NVIDIA H100/H200/Blackwell, AMD MI300/MI325/MI350, Google TPU v5p/v6, Intel Gaudi 3, Cerebras WSE-3.

## 5. 인터커넥트 (가속기)

| 이름 | 속도 | 용도 |
| --- | --- | --- |
| **NVLink 5** (Blackwell) | 1.8 TB/s/GPU | 가속기 간 직결 |
| **NVSwitch** | rack 단 fat-tree | 72~576 GPU |
| **AMD Infinity Fabric 4** | — | 코어 / IO / GPU |
| **UALink / Ultra Ethernet** | (예정) | NVIDIA 독점 대안 컨소시엄 |

## 6. NUMA / Chiplet

대형 SoC 는 칩렛 / 다수 다이 — NUMA 영향:
- Apple M2/M3 Ultra = 두 다이.
- AMD EPYC = N CCD + 1 IOD.
- 같은 다이 메모리가 가장 빠름. 다른 다이 / 다른 소켓 메모리는 latency 1.5~3 배.

## 7. 함정

1. **UMA = 모든 가속기 동시 사용** 으로 오해 — 같은 DRAM 풀이지만 BW 는 유한.
2. **NPU TOPS 직접 비교** — INT8 vs FP16 vs INT4 의 TOPS 가 모두 다른 정밀도.
3. **NUMA 무시한 컨테이너 배치** — 한 코어 / 한 vCPU 가 cross-socket 메모리 접근으로 latency 폭증.
4. **Apple Silicon = 영원히 호환** 가정 — 같은 OS 라도 메이저 업그레이드마다 ABI 미세 변경.

## 8. 관련

- [[../memory/ddr-evolution]] — HBM / LPDDR / UMA
- [[../bus-io/pcie]] — NVLink / Infinity Fabric / UCIe
- [[../transistor/process-node-evolution]] — 첨단 공정의 실 대표 적용처
