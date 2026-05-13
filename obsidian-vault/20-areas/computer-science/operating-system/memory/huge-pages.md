---
title: "Huge Pages — 2MB / 1GB"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T12:10:00+09:00
tags:
  - operating-system
  - memory
  - huge-page
---

# Huge Pages

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | THP / explicit / 사용 사례 |

**[[memory|↑ Memory hub]]**

---

## 1. 한 줄

기본 4KB 페이지 대신 **2 MB / 1 GB** 페이지 사용 → TLB / 페이지 테이블 비용 절약.

---

## 2. 효과

```
4 KB:  TLB 64 entry × 4 KB = 256 KB 커버
2 MB:                       128 MB 커버 (512배)
1 GB:                       64 GB 커버
```

→ TLB miss 폭감. DB / JVM / 큰 메모리 워크로드 효과 ↑.

또한 4-level → 3-level (PT level skip) → page walk 비용 ↓.

---

## 3. 두 종류

### 3.1 Explicit HugeTLB

명시적으로 huge page reserve + 사용:

```bash
echo 1000 > /proc/sys/vm/nr_hugepages       # 2 MB × 1000

cat /proc/meminfo | grep Huge
# HugePages_Total:    1000
# HugePages_Free:     1000
# Hugepagesize:       2048 kB
```

```c
mmap(NULL, sz, PROT_READ|PROT_WRITE,
     MAP_PRIVATE|MAP_ANONYMOUS|MAP_HUGETLB, -1, 0);
```

또는 hugetlbfs 마운트.

### 3.2 THP (Transparent Huge Page)

자동. 커널이 4 KB 페이지 묶어 2 MB 으로 합침.

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
# [always] madvise never
```

| 값 | 의미 |
| --- | --- |
| `always` | 가능하면 항상 |
| `madvise` | `MADV_HUGEPAGE` 명시 시만 |
| `never` | off |

---

## 4. 사용 사례

| 시스템 | huge page |
| --- | --- |
| HotSpot JVM | `-XX:+UseLargePages` |
| PostgreSQL | `huge_pages = try / on / off` |
| Oracle | shm + hugepage |
| MySQL InnoDB | `large_pages` |
| Redis | THP 비추 (fork 시 latency) |
| MongoDB | THP 비추 |
| KVM / DPDK | 1 GB 활용 |

---

## 5. PostgreSQL 예

```ini
huge_pages = try   # 또는 on
shared_buffers = 8GB
```

```bash
# 필요 huge page 수 계산
echo "huge pages needed = shared_buffers / Hugepagesize"
echo 4096 > /proc/sys/vm/nr_hugepages
```

---

## 6. JVM

```bash
-XX:+UseLargePages
-XX:LargePageSizeInBytes=2m
```

reserve 필요. K8s 의 hugepage 자원도 가능 (`hugepages-2Mi` resource).

---

## 7. THP 의 함정 ⚠️

### 7.1 Latency Spike

- THP 가 새 huge page 만들려면 4 KB 페이지 512 개 contiguous 가 필요
- compaction (defrag) 이 동기 발생 시 ms 단위 stall
- DB / Redis 의 fork() + COW + THP = 매우 느림

→ DB 의 흔한 권장: **THP off** (또는 madvise).

### 7.2 Memory 폭증
4 KB 만 필요한데 2 MB 통째로 → RSS ↑.

### 7.3 안 깨지는 통째 swap-out
2 MB swap-out / in.

### 7.4 madvise 모드 권장

```bash
echo madvise > /sys/kernel/mm/transparent_hugepage/enabled
echo madvise > /sys/kernel/mm/transparent_hugepage/defrag
```

명시한 영역만 THP. 안전.

---

## 8. 1 GB Huge Page

x86-64 의 PDPT level. boot param 으로 reserve:

```
default_hugepagesz=1G hugepagesz=1G hugepages=16
```

→ 부팅 시 16 GB reserve. 변경 불가.

KVM / DPDK / 거대 in-memory DB.

---

## 9. /sys 확인

```bash
ls /sys/kernel/mm/hugepages/
# hugepages-2048kB/  hugepages-1048576kB/

cat /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
cat /sys/kernel/mm/hugepages/hugepages-2048kB/free_hugepages
```

---

## 10. 모니터링

```bash
cat /proc/meminfo | grep -E 'Huge|AnonHuge|ShmemHuge'

# THP 의 RSS
cat /proc/$PID/smaps | grep -i hugepage

# defrag 폐기
grep thp_fault_alloc /proc/vmstat
grep thp_split /proc/vmstat
```

---

## 11. NUMA + Huge Page

특정 node 에 reserve:

```bash
echo 100 > /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
```

→ NUMA-aware app 의 효율.

---

## 12. K8s 의 hugepage

```yaml
resources:
  limits:
    hugepages-2Mi: 1Gi
    memory: 2Gi
  requests:
    hugepages-2Mi: 1Gi
    memory: 2Gi
```

노드에 미리 reserve 되어 있어야.

---

## 13. 함정 (요약)

### 13.1 THP always 의 latency spike
DB / Redis = madvise / never.

### 13.2 1 GB huge page boot param 필수
런타임 변경 X.

### 13.3 NUMA mismatch
다른 node 의 hugepage 사용 시 cross-node latency.

### 13.4 측정 없이 결정
일부 워크로드는 효과 X / 손해.

### 13.5 explicit hugetlbfs 의 lock 비호환
swap 안 됨. memory 부족 시 fail.

### 13.6 fork + THP + write
COW 폭증 — Redis BGSAVE 의 흔한 함정.

---

## 14. 학습 자료

- **kernel.org** `Documentation/admin-guide/mm/hugetlbpage.rst`
- `Documentation/admin-guide/mm/transhuge.rst`
- **Brendan Gregg — THP 시리즈**
- PostgreSQL / MongoDB / Redis 의 huge page 권장 문서

---

## 15. 관련

- [[tlb]]
- [[page-table]]
- [[mmap]]
- [[memory]] — Memory hub
