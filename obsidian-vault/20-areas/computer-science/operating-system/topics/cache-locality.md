---
title: "Cache Locality — 캐시 친화 코드"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:40:00+09:00
tags:
  - operating-system
  - topic
  - cache
---

# Cache Locality

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | locality 원칙 + 측정 |

**[[topics|↑ Topics hub]]**

---

## 1. 한 줄

CPU 캐시 (L1/L2/L3) 의 hit rate 가 성능의 핵심.
"가까운 메모리는 자주, 한 번 접근하면 그 주변도".

```
L1 cache: ~ 1 ns
L2:       ~ 4 ns
L3:       ~ 10 ns
RAM:      ~ 100 ns      ← 10-100배 차이
```

자세히 → [[../memory/memory#2-메모리-계층]]

---

## 2. 두 종류

| Locality | 의미 |
| --- | --- |
| **Temporal** | 한 번 접근한 데이터 = 곧 다시 접근 |
| **Spatial** | 한 위치 접근 후 = 근처도 접근 |

→ 캐시는 둘 다 활용 — 64 B cache line 단위로 fetch + LRU.

---

## 3. Cache Line — 64 byte

```
한 cache miss = 64 byte (또는 128 byte) fetch
→ 64 byte 안의 모든 byte 가 cache 에 들어옴
→ sequential access = miss 1/16 (int 16 개)
→ random access = 매번 miss
```

→ **sequential / linear 자료구조** 가 압도적 빠름.

---

## 4. 배열 vs 연결 리스트

```c
// array — cache 친화
int arr[N];
for (i=0; i<N; i++) sum += arr[i];     // 매 16 개 마다 miss

// linked list — cache 적
struct Node { int v; Node *next; };
Node *p = head;
while (p) { sum += p->v; p = p->next; }   // 매번 miss + pointer chase
```

→ 같은 N 에 10-100배 차이.

---

## 5. AoS vs SoA

```c
// Array of Structs
struct Particle { float x, y, z; float vx, vy, vz; };
Particle ps[N];

// Struct of Arrays
struct Particles { float x[N], y[N], z[N], vx[N], vy[N], vz[N]; };
```

x 만 처리 시:
- AoS = 24 byte stride → 매 cache line 의 일부
- SoA = sequential → 모든 byte 사용

→ HPC / game / SIMD 에서 SoA 가 우위.

---

## 6. Row-major vs Column-major

```c
// row-major (C, default)
for (i=0; i<N; i++)
  for (j=0; j<N; j++)
    sum += a[i][j];                       // 같은 row → sequential

// 잘못된 (cache miss)
for (j=0; j<N; j++)
  for (i=0; i<N; i++)
    sum += a[i][j];                       // 같은 column → stride N
```

같은 알고리즘에 10x+ 차이.

Fortran / NumPy = column-major (default).

---

## 7. 자료구조 trade-off

| 자료구조 | locality |
| --- | --- |
| array / vector | ★★★★★ |
| open-addressing hash | ★★★★ |
| binary heap | ★★★ |
| B-tree (high fanout) | ★★★ |
| balanced BST (red-black, AVL) | ★ |
| linked list | ★ |
| chained hash | ★★ |

→ **flat / contiguous = 빠름**.

---

## 8. 단순 알고리즘이 빠른 이유

```
linear search on 1000 element array
vs
binary search on linked list (불가) / 또는 sorted array
```

- N < 100 = linear scan 압승 (cache 친화)
- N > 1000 = binary search 우위 (단, cache friendly array 에서)

→ 작은 N 의 자료구조 선택 = **cache friendly** 우선.

---

## 9. False Sharing

```c
struct {
    atomic_int a;     // thread 1
    atomic_int b;     // thread 2
} pad;
```

a, b 가 같은 64 byte cache line → 한 코어가 a 쓰면 다른 코어의 b cache invalidate → bouncing.

해결:
```c
struct {
    atomic_int a;
    char _pad[60];
    atomic_int b;
} __attribute__((aligned(64)));
```

자세히 → [[../synchronization/atomic#12-false-sharing]]

---

## 10. NUMA + cache

다른 NUMA node 의 데이터 → remote cache miss → 3x 느림.

```
local L1:  1 ns
remote L1: 100+ ns (effectively cache miss)
```

해결 → [[../memory/numa]]

---

## 11. TLB Locality

페이지 (4 KB) 단위 mapping. 작은 working set 이면 TLB 도 hit.

```
TLB entries: ~64 (L1) ~1500 (L2)
4 KB × 64 = 256 KB coverage
```

큰 working set + 4 KB 페이지 = TLB miss 폭증.

해결: **Huge Page** (2 MB / 1 GB) — TLB coverage 폭증.
자세히 → [[../memory/huge-pages]]

---

## 12. 측정

```bash
# 캐시 / TLB stat
sudo perf stat -e cycles,instructions,\
cache-references,cache-misses,\
L1-dcache-loads,L1-dcache-load-misses,\
dTLB-loads,dTLB-load-misses \
./app

# 분석
# IPC = instructions / cycles → > 1.0 좋음
# cache miss % = cache-misses / cache-references
# TLB miss % = dTLB-misses / dTLB-loads
```

자세히 → [[perf]], [[../linux/strace-perf]]

---

## 13. 실전 권장

### 13.1 데이터를 함께 두기
같이 쓰는 field → 같은 struct.

### 13.2 작은 자료
hot 자료 = cache fit. 큰 자료는 streaming.

### 13.3 사전 정렬
sorted array > BST. cache 친화.

### 13.4 prefetch
```c
__builtin_prefetch(ptr, 0, 3);     // 미리 fetch
```

CPU 의 자동 prefetcher 가 보통 충분 — 명시는 특수 케이스.

### 13.5 SIMD + SoA
vector instruction (AVX) = 정렬된 array 에 최적.

### 13.6 padding / alignment
hot atomic counter / mutex = cache line 정렬.

---

## 14. 응용

| 시스템 | locality 활용 |
| --- | --- |
| LMDB / B-tree DB | 큰 fanout — 한 페이지에 많은 키 |
| ClickHouse | column-oriented — 같은 column 의 데이터 sequential |
| Folly F14 | open-addressing hash |
| FoundationDB | sorted KV |
| ScyllaDB / Seastar | per-CPU, NUMA-aware |
| Rust std::HashMap | 새 backend (hashbrown) — SwissTable, cache friendly |

---

## 15. 함정

### 15.1 "Big O 만 보고"
O(log N) 의 BST > O(N) array → 작은 N 에서 array 가 빠름 (cache).

### 15.2 false sharing 무시
hot atomic 코어 별 — bouncing 폭증.

### 15.3 NUMA 무시
cross-node = remote miss.

### 15.4 random access on 큰 hash
hash 의 random nature 가 cache 친화 X.

### 15.5 prefetch 남용
잘못된 prefetch = cache pollution.

### 15.6 cache size 가정
L1 32 KB, L2 256 KB - 1 MB, L3 수십 MB. CPU 마다 다름.

### 15.7 PE 코어 vs E 코어 (Intel hybrid)
다른 cache 구조. 측정 필수.

---

## 16. 학습 자료

- **What Every Programmer Should Know About Memory** — Ulrich Drepper (필독)
- **Computer Systems: A Programmer's Perspective** (CSAPP)
- **Latency Numbers Every Programmer Should Know** — Jeff Dean
- **Brendan Gregg — Cache Studies**

---

## 17. 관련

- [[../memory/memory]]
- [[../memory/tlb]]
- [[../memory/huge-pages]]
- [[../memory/numa]]
- [[../synchronization/atomic]]
- [[topics]] — Topics hub
