---
title: "메모리 계층 (Memory Hierarchy)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, memory, hierarchy, cache, latency, bandwidth, sram, locality, tlb, prefetch, mesi]
---

# 메모리 계층 (Memory Hierarchy)

**[[memory|↑ 메모리]]**

> 현대 시스템 성능의 진짜 천장 — CPU 클럭이 아니라 캐시 미스 / DRAM latency.

## 1. 계층 전체 표

| 단계 | 용량 | latency | bandwidth | 매체 | 가격 (GB 당) |
| --- | --- | --- | --- | --- | --- |
| Register | KB | < 1 ns | TB/s | SRAM in core | — |
| **L1 cache** | 32–64 KB/core | 1–4 cycle (~1 ns) | TB/s | SRAM | 매우 비쌈 |
| **L2 cache** | 256 KB–2 MB/core | 10–15 cycle | hundreds GB/s | SRAM | 비쌈 |
| **L3 / LLC** | 8–384 MB/chip | 30–60 cycle (~10–20 ns) | hundreds GB/s | SRAM | 비쌈 |
| DRAM | 16 GB–4 TB | ~80–100 ns | 50–100 GB/s/channel | DRAM | $3–8 |
| CXL.mem | TB+ | 200–300 ns | PCIe 위 | DRAM/PMM | 비슷 |
| NVMe SSD | 0.5–60 TB | 10–100 μs | 3–14 GB/s | NAND | $0.05–0.2 |
| Network (10 GbE) | — | 0.1–1 ms | 1.25 GB/s | NIC | — |
| HDD | 1–24 TB | 5–15 ms | 100–200 MB/s | platter | $0.01–0.03 |
| Tape (LTO-9) | 18 TB | seconds | 400 MB/s | magnetic | $0.005 |

### 핵심 관찰
- 한 단 내려갈 때마다 **latency ~10 배**.
- L3 → DRAM 한 번 미스 = 1 시간 작업이 80 시간 걸리는 셈.
- DRAM → NVMe 한 번 미스 = 1 분이 1 일.
- NVMe → 네트워크 미스 = 1 일이 30 일.

→ **캐시 적중률 1% 차이가 실효 throughput 의 천장**.

## 2. 왜 계층이 필요한가 — Locality 원리

### 시간적 지역성 (Temporal Locality)
- 최근에 접근한 데이터를 또 곧 접근할 가능성 ↑.
- 변수 / loop counter / 함수 인수.

### 공간적 지역성 (Spatial Locality)
- 한 주소 접근 후 가까운 주소 접근 가능성 ↑.
- 배열 sequential / instruction stream / struct field.

→ 캐시가 cache line (보통 64 byte) 단위로 가져옴. 인접 데이터 자동 prefetch.

## 3. SRAM 셀 동작

```
       Vdd                       Vdd
        │                         │
       pMOS    cross-coupled     pMOS
        │ ────────────────────── │
       nMOS                      nMOS
        │                         │
       GND                       GND
        │                         │
       BL                        BLB   ← differential bit lines
```

### 6-T 셀
- pMOS 2 + nMOS 2 = cross-coupled 인버터 두 개.
- pass transistor 2 (word line 으로 enable).
- 두 인버터가 서로 출력을 입력으로 → 0/1 안정점.
- 전원 있는 한 까먹지 않음.

### Read
- WL = HIGH → pass transistor ON.
- 셀의 0/1 에 따라 BL / BLB 미세 전압 차이.
- sense amp 가 증폭 → 0/1 결정.

### Write
- BL / BLB 에 강제 0 / 1.
- WL HIGH → pass transistor 가 외부 강제 신호를 셀에 주입.
- 인버터 쌍 안정점 flip.

### 면적
- SRAM = 6 트랜지스터 / bit.
- DRAM = 1 트랜지스터 + 1 캐패시터 / bit.
- SRAM 면적이 DRAM 의 ~10 배 → 그래서 캐시는 비쌈 + 작음.

## 4. 캐시 동작 — Line / Tag / Index / Offset

### Address 분해

예: 64-bit 주소, 64 byte line, 256 KB 8-way L2.

```
   ┌────────────────────────────────────┬─────────┬───────────┐
   │  Tag (남은 비트)                    │ Index   │ Offset    │
   └────────────────────────────────────┴─────────┴───────────┘
                                          9 bit      6 bit
```

- **Offset (6 bit)**: 64-byte line 안 위치.
- **Index (9 bit)**: 512 set 중 어느 set 인지. 256 KB / 64 byte / 8 way = 512 set.
- **Tag (남은 49 bit)**: cache 안 line 식별.

### 동작
1. 주소 → index 로 set 찾음.
2. 그 set 안 8 way 의 tag 모두 비교.
3. 일치 + valid → hit (그 way 의 data 반환).
4. 다 miss → 다음 단 캐시 / DRAM 으로.

## 5. 캐시 구성 — Associativity

| 종류 | 설명 | 장단점 |
| --- | --- | --- |
| **Direct-mapped** | 한 주소 → 정확히 1 line | 단순 / 빠름. conflict miss 심함. |
| **Fully associative** | 어디든 가능 | conflict miss 0. 하드웨어 비쌈. |
| **N-way set associative** | set 안 N 위치 가능 | 균형. 거의 모든 현대 캐시. |

전형:
- L1: 4-8 way.
- L2: 8-16 way.
- L3 / LLC: 16-32 way.

## 6. Cache 의 3 가지 미스

### Compulsory (Cold) miss
- 처음 접근. 어떤 캐시도 못 막음.
- prefetch 만 도움.

### Capacity miss
- 워킹셋 > 캐시 크기.
- 캐시 크기 늘리거나 워킹셋 줄여야.

### Conflict miss
- set 안 way 부족으로 동일 set 의 다른 line 이 evict.
- associativity ↑ 또는 line 배치 변경.

## 7. Write 정책

### Write-Through
- write 즉시 다음 단까지 전파.
- 단순. data consistency 강함.
- 단점: write bandwidth 폭주.

### Write-Back
- write 는 현 캐시에만. dirty bit 설정.
- evict 시 다음 단으로 write.
- 거의 모든 현대 CPU 의 L1 / L2 / L3.

### Write-Allocate vs No-Write-Allocate
- write miss 시 line 을 캐시에 로드 vs 안 함.
- write-allocate 가 일반.

## 8. Replacement 정책

| 정책 | 의미 |
| --- | --- |
| **LRU (Least Recently Used)** | 가장 오래 안 쓴 line evict. 이상적이지만 N-way 구현 비쌈. |
| **Pseudo-LRU (PLRU)** | tree 구조로 근사. 8-way 이상 표준. |
| **Random** | 랜덤. 구현 단순. |
| **NRU (Not Recently Used)** | 1-bit per line. |
| **DRRIP / RRIP** | Re-Reference Interval Prediction. modern. |

Intel / AMD 현대 LLC = adaptive replacement (워크로드 따라 알고리즘 변경).

## 9. MESI / MOESI — Coherence 프로토콜

멀티 코어 사이 캐시 일관성.

### MESI 4 상태
| 상태 | 의미 |
| --- | --- |
| **M (Modified)** | 나만 dirty 보유. main memory stale. |
| **E (Exclusive)** | 나만 clean 보유. main memory 와 일치. |
| **S (Shared)** | 여러 코어가 clean 보유. |
| **I (Invalid)** | 무효. |

### MOESI (AMD) — O 추가
- **O (Owned)** — dirty 면서 다른 코어와 공유. owner 가 evict 시 write-back.

### Snoop / Directory
- **Snoop**: bus 위 모든 코어가 read/write 트래픽 보고 자기 캐시 상태 갱신.
- **Directory**: 중앙 / 분산 directory 가 각 line 의 owner 추적. 큰 멀티 코어에서 scalable.

### False Sharing — 가장 흔한 성능 문제
- 두 코어가 다른 변수지만 같은 64-byte cache line 공유.
- 각 코어가 자기 변수 write 마다 상대 캐시 invalidate.
- 코드 상 race 없어도 성능 1/10.

```c
// 나쁨
struct counter {
    long a;     // CPU 0 이 빈번 update
    long b;     // CPU 1 이 빈번 update
};                // 같은 64-byte line 안

// 좋음
struct counter {
    long a;
    char _pad[56];   // 채워서 b 를 다른 line 으로
    long b;
};

// 또는 C++17
struct counter {
    alignas(64) long a;
    alignas(64) long b;
};
```

## 10. Prefetching

### Hardware Prefetcher
- CPU 가 stride 패턴 감지 → 다음 line 미리 가져옴.
- L1 hardware prefetcher: stride / IP-based.
- L2 streamer.
- Intel 의 PSF / PSP 등.

### Software Prefetch
```c
__builtin_prefetch(&array[i + 16], 0, 3);   // GCC
_mm_prefetch(&array[i + 16], _MM_HINT_T0);   // Intel intrinsic
```

- 효과: latency hiding.
- 부작용: 잘못 쓰면 캐시 오염.
- 일반 코드는 hardware prefetcher 가 거의 다 잡음.

## 11. TLB — Virtual → Physical 매핑 캐시

가상 → 실제 주소 변환은 page table walk 필요 (4-level 의 경우 4 회 메모리 접근). 너무 비쌈 → TLB 가 최근 변환 캐시.

### 구조
- L1 ITLB / DTLB: 64-128 entry.
- L2 STLB: 1024-4096 entry.
- ASID (Address Space ID): 프로세스 별 TLB flush 줄임.

### TLB Miss
- page table walk 필요. 4-level 의 경우 4 회 메모리 접근 (캐시에서 일부 hit).
- 1 miss = ~100 ns 영향.

### Huge Page
- 4 KB → 2 MB / 1 GB page.
- 같은 TLB entry 가 더 큰 영역 cover.
- DB / VM / ML 워크로드의 표준 최적화.
- Linux Transparent Huge Page (THP) 또는 explicit hugetlbfs.

## 12. 캐시 친화적 코드 — AoS vs SoA

### AoS (Array of Structures)
```c
struct Particle { float x, y, z, m; };
Particle particles[N];
```
- 모든 field 가 같은 line 에 있음. 모든 field 를 함께 쓰면 좋음.
- 하지만 x 만 sequential 처리 시 나쁨 (4 배 메모리 traffic).

### SoA (Structure of Arrays)
```c
struct ParticleArr {
    float x[N], y[N], z[N], m[N];
};
```
- 같은 field 들이 연속 → SIMD / vector 친화.
- ML / graphics / physics 표준 패턴.

### Hot/Cold split
- struct 의 자주 쓰는 field 만 따로 묶기.
- 캐시 적중 ↑.

## 13. 성능 측정 / 진단

```bash
# 캐시 크기
lscpu | grep -i cache

# 캐시 미스 카운터
sudo perf stat -e cache-misses,cache-references,\
LLC-load-misses,LLC-store-misses,\
L1-dcache-load-misses,L1-dcache-loads ./mybin

# Branch + memory + IPC
sudo perf stat -e cycles,instructions,branches,branch-misses,\
cache-references,cache-misses ./mybin

# cache-to-cache (false sharing 등)
sudo perf c2c record ./mybin
sudo perf c2c report

# memory bandwidth 측정
sudo apt install likwid
likwid-bench -t copy -w S0:8GB
likwid-bench -t stream_avx512 -w S0:8GB

# Intel Memory Latency Checker
sudo ./mlc --latency_matrix
sudo ./mlc --bandwidth_matrix
sudo ./mlc --idle_latency

# TLB miss
sudo perf stat -e dTLB-load-misses,iTLB-load-misses ./mybin
```

## 14. 캐시 미스의 실 영향 — 예제

64 byte line, 30 ns L3→DRAM 미스, 3 GHz CPU 가정.

### 예 1 — Sequential read
- `for (int i = 0; i < N; i++) sum += a[i];`
- prefetcher 가 잘 동작 → cache hit 99%+.
- Throughput = 메모리 BW (10-50 GB/s).

### 예 2 — Random read (큰 hash table lookup)
- 매 lookup 마다 다른 line.
- 워킹셋 > LLC → 매 lookup 이 ~80-100 ns.
- Throughput = 10-12 M lookup/s per core.

### 예 3 — Pointer chasing (linked list)
- 한 노드 read 후에야 다음 노드 주소 알 수 있음.
- prefetch 불가.
- 각 노드 = 1 cache miss = 80-100 ns.
- 1000 노드 traverse = 80-100 μs.

→ 같은 코드 보이는 두 알고리즘이 캐시 미스 패턴 때문에 100× 성능 차이.

## 15. NUMA 와 메모리 계층

자세히: [[../soc/soc-anatomy]] 의 NUMA 섹션.

- 같은 NUMA 노드 메모리: 80-100 ns.
- cross-NUMA 메모리: 130-250 ns.
- 캐시는 보통 NUMA 노드 별 분리 (각 소켓 자기 LLC).
- 한 메모리 영역을 두 NUMA 의 코어가 자주 read/write → cross-socket invalidate 폭주.

```bash
numactl -H                              # 토폴로지
numactl --membind=0 --cpunodebind=0 ./app   # 로컬 메모리만
```

## 16. CXL.mem — 새 메모리 티어

자세히: [[ddr-evolution]] §5.

- DRAM 200-300 ns 정도 latency.
- 일반 DDR 의 2-3 배.
- 하지만 NVMe (μs 단위) 보다 100-1000 배 빠름.
- **새로운 티어**: hot=DRAM, warm=CXL.mem, cold=NVMe.

## 17. 실 사례 — 캐시 의식 최적화

### 사례 1 — Hash table size 가 LLC 안에 들어가게
- LLC = 32 MB 인데 hash table = 50 MB → DRAM bound.
- 24 MB 로 줄이면 throughput 3 배 ↑.

### 사례 2 — Loop tiling (Matrix multiply)
```c
// 나쁨
for (i) for (j) for (k) C[i][j] += A[i][k] * B[k][j];

// 좋음 — tile B 가 캐시 안에 머무름
for (ii) for (jj) for (kk)
  for (i in tile) for (j in tile) for (k in tile)
    C[i][j] += A[i][k] * B[k][j];
```
- BLAS gemm 의 표준 기법.
- 캐시 적중률 ↑↑.

### 사례 3 — Concurrent counter false sharing
- 100 코어 가 각자 self counter += 1.
- 모두 같은 64-byte line → cache line ping-pong.
- per-core padding 으로 throughput 100× ↑.

## 18. 캐시 attack — 사이드채널

- **Spectre / Meltdown (2018)**: speculative execution 이 캐시에 잔여 데이터 남김.
- **Flush+Reload** — 공격자가 victim 의 캐시 line 측정.
- **Prime+Probe** — set 채워두고 victim 동작 후 비교.
- 대응: 캐시 partitioning (Intel CAT), kernel page table isolation (KPTI).

자세히: [[../../security-theory/security-theory]].

## 19. 함정

1. **"L3 크면 무조건 빠르다"** — 워킹셋이 L2 안이면 무의미.
2. **TLB miss 무시** — 캐시 미스만큼 비쌈. huge page 도움.
3. **False sharing** — 동시성 변수를 같은 line 에 두면 100× ↓.
4. **prefetch 강제 사용** — 잘못 쓰면 캐시 오염. hardware prefetcher 가 거의 다 잡음.
5. **AoS vs SoA 무관 가정** — sequential workload 에서 4 배 차이.
6. **NUMA 무시** — cross-socket 메모리로 latency 폭증.
7. **캐시 partitioning 활성** — 클라우드 host 의 noisy neighbor 격리에 사용. 기본은 OFF.
8. **L1 / L2 / L3 의 inclusive / exclusive 정책 혼동** — Intel L2/L3 = inclusive 일부, AMD = mostly exclusive.

## 20. 관련

- [[memory]]
- [[dram-cell]] — DRAM 의 latency 가 이 계층의 천장
- [[ddr-evolution]] — channel BW
- [[ecc-rowhammer]] — DRAM 안정성
- [[../../computer-architecture/computer-architecture]] — MESI / 분기 / pipeline / OOO
- [[../../operating-system/operating-system]] — virtual memory / page / TLB / huge page
- [[../../security-theory/security-theory]] — Spectre / Meltdown / 캐시 사이드채널
