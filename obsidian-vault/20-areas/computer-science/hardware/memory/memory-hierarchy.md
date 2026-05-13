---
title: "메모리 계층 (Memory Hierarchy)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, memory, hierarchy, cache, latency, bandwidth, sram]
---

# 메모리 계층 (Memory Hierarchy)

**[[memory|↑ 메모리]]**

## 1. 계층 표

| 단계 | 용량 | latency | bandwidth (단일) | 매체 |
| --- | --- | --- | --- | --- |
| Register | ~KB | < 1 ns | TB/s | SRAM in core |
| L1 cache | 32–64 KB/core | 1–4 cycle (~1 ns) | TB/s | SRAM |
| L2 cache | 256 KB–2 MB/core | 10–15 cycle | hundreds GB/s | SRAM |
| L3 / LLC | 8–384 MB/chip | 30–60 cycle | hundreds GB/s | SRAM |
| DRAM | 16 GB–4 TB | ~80–100 ns | 50–100 GB/s/channel | DRAM |
| NVMe SSD | 0.5–60 TB | 10–100 μs | 3–14 GB/s | NAND |
| Network (10 GbE) | — | 0.1–1 ms | 1.25 GB/s | NIC |
| HDD | 1–24 TB | 5–15 ms | 100–200 MB/s | platter |
| Tape (LTO-9) | 18 TB | seconds | 400 MB/s | magnetic |

> 한 단 내려갈 때마다 latency ~10 배. **L3 → DRAM 한 번의 미스 = 1 시간 작업이 80 시간 걸리는 셈.**

## 2. SRAM 셀 동작

6 트랜지스터: pMOS 2 + nMOS 2 (cross-coupled inverter) + pass gate 2.

```
       Vdd                       Vdd
        │                         │
       pMOS    cross-coupled     pMOS
        │ ────────────────────── │
       nMOS                      nMOS
        │                         │
       GND                       GND
        │                         │
       BL                        BLB   ← bit lines
```

- 두 인버터가 서로의 출력을 입력으로 → 0/1 둘 중 한 안정점.
- 전원 있는 한 안 까먹음.
- 빠르지만 1 bit 당 면적이 DRAM 의 ~10 배.

## 3. 캐시 line / 매핑 / 일관성 — 짧게

자세히는 [[../../computer-architecture/computer-architecture]] 의 캐시 섹션.

- **line size** — 보통 64 byte. 메모리 접근 단위.
- **set associativity** — direct-mapped / N-way / fully associative.
- **replacement** — LRU / pseudo-LRU / random.
- **write policy** — write-through / write-back.
- **coherence** — MESI / MOESI. 멀티 코어 사이 일관성.

## 4. 지표 / 진단

```bash
lscpu | grep -i cache       # L1/L2/L3 크기
perf stat -e cache-misses,cache-references,LLC-load-misses ./bin
perf c2c                    # cache-to-cache 데이터 이동 분석
```

응용:
- **워킹셋이 L2 안** → 가속기 / SIMD 의 BW 가 살아남.
- **DRAM bound** → AoS → SoA 변환, prefetch hint, NUMA pinning.

## 5. 데이터 레이아웃 — 같은 코드, 다른 성능

- **AoS (Array of Structures)** vs **SoA (Structure of Arrays)** — 캐시 라인을 의미 있게 채우는 쪽이 훨씬 빠름.
- **hot/cold split** — 자주 쓰는 필드만 한 구조체로 분리해 캐시 적중 ↑.
- **false sharing** — 두 코어가 다른 변수지만 같은 64-byte line 을 공유 → cache invalidate 폭주.

## 6. 함정

1. **"L3 가 크면 무조건 빠르다"** — 워킹셋이 L2 안에 다 들어가면 L3 크기 무의미.
2. **TLB miss 무시** — 가상 → 물리 변환 캐시 (TLB) 미스도 캐시 미스만큼 비쌈. huge page 가 도움.
3. **false sharing** — 동시성 변수 두 개를 같은 cache line 에 두면 2 배 ↓.

## 7. 관련

- [[memory]]
- [[dram-cell]]
- [[../../computer-architecture/computer-architecture]] — MESI / 분기 / TLB
