---
title: "TLB — Translation Lookaside Buffer"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T11:45:00+09:00
tags:
  - operating-system
  - memory
  - tlb
---

# TLB

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | TLB / shootdown / huge page |

**[[memory|↑ Memory hub]]**

---

## 1. 한 줄

가상→물리 변환 결과의 **CPU 내장 캐시**. hit 시 page walk 없음 (0 cycle).

---

## 2. 구조

```
[VPN] → [PFN + flags]   (associative cache, ~64-2048 entry)
L1 TLB: 64 entry, ~ 1 cycle hit
L2 TLB: 1500 entry, ~ 7 cycle hit
miss → page walk (수십 ~ 100 cycles)
```

- I-TLB (instruction) / D-TLB (data) 분리
- L1 / L2 분리
- HT / SMT 코어가 공유

---

## 3. Page Walk 비용

```
TLB hit:   0-1 cycle
TLB miss + page walk:
  4-level x86-64 → 4 메모리 access
  각 ~ 100 ns (L3 miss / DRAM 가정)
  → 400 ns ~ 1 μs
```

→ TLB hit rate 가 성능에 큰 영향.

---

## 4. ASID / PCID — Process Context ID

```
TLB entry: [VPN, PCID] → [PFN, flags]
```

- context switch 시 TLB flush 안 함 (다른 PCID 그대로)
- Linux 4.14+ + Intel Westmere+

→ context switch 비용 ↓.

KPTI (Meltdown 패치) 의 부담 완화.

---

## 5. TLB Shootdown

```
프로세스 A 가 mmap 변경 → PTE 무효화
모든 CPU 의 TLB 에서 그 VPN 제거 필요
IPI (inter-processor interrupt) 로 모든 CPU 에 명령
```

수십 ~ 수백 코어 = shootdown 비용 폭증.

→ munmap / mprotect 가 잦으면 hot spot.

---

## 6. Huge Page 의 TLB 효과

```
4 KB 페이지 64 entry TLB = 256 KB coverage
2 MB Huge Page = 128 MB coverage
1 GB Huge Page = 64 GB coverage
```

→ TLB miss 폭감. DB / JVM / 대용량 워크로드에 효과적.

자세히 → [[huge-pages]]

---

## 7. 측정

### 7.1 perf

```bash
perf stat -e dTLB-loads,dTLB-load-misses,iTLB-loads,iTLB-load-misses ./app

# miss rate
```

### 7.2 perf list

```bash
perf list | grep -i tlb
```

CPU 마다 다른 event.

### 7.3 vmstat / cpustat
TLB 자체 stat 은 없음. miss rate 가 핵심 지표.

---

## 8. 줄이는 방법

### 8.1 Huge Page
가장 큰 효과.

### 8.2 Locality of reference
같은 페이지 안에서 접근. 큰 hash map / 무작위 접근 = TLB 폭증.

### 8.3 NUMA pinning
[[numa]] — cross-node TLB 영향.

### 8.4 PCID enabled CPU
context switch 비용 ↓.

### 8.5 큰 작업의 mmap shootdown 회피
mmap 통째로 + 점진적 사용 (lazy fault).

---

## 9. JVM / DB 의 활용

| 시스템 | Huge Page 사용 |
| --- | --- |
| **HotSpot JVM** | `-XX:+UseLargePages` |
| **PostgreSQL** | `huge_pages = try` |
| **Oracle** | shm 으로 huge page |
| **MongoDB** | THP 비추 (smaps 표기 권장) |

THP (Transparent Huge Page) 는 fragmentation / latency spike → DB 는 보통 끔 + explicit huge page 사용.

---

## 10. PMD / PUD / PML4 의 TLB

multi-level page walk 에서 상위 단계의 entry 도 TLB 비슷한 캐시 (Paging-Structure Cache).

→ Huge Page 가 상위 cache 도 활용 → 효과 ↑.

---

## 11. ARM 의 TLB

ARM = ASID 표준 (PCID 와 비슷). `dsb ish` / `tlbi` 명령.

---

## 12. 함정

### 12.1 큰 hash table 의 무작위 접근
TLB miss 폭증. Huge Page or 다른 자료구조.

### 12.2 THP 의 fragmentation
Page 가 fragmented → THP allocation fail → defrag (느림).

### 12.3 잦은 mmap/munmap
TLB shootdown.

### 12.4 측정 안 하고 huge page 활성
일부 워크로드는 오히려 손해 (메모리 ↑, init 시 huge page allocation 비용).

### 12.5 NUMA cross-node
TLB 와 cache 모두 cold.

---

## 13. 학습 자료

- **OSTEP** Ch. 19
- **What Every Programmer Should Know About Memory** — Drepper
- **Intel SDM Vol. 3 Ch. 4.10** (TLB)
- **brendangregg.com — TLB 시리즈**

---

## 14. 관련

- [[page-table]]
- [[huge-pages]]
- [[numa]]
- [[memory]] — Memory hub
