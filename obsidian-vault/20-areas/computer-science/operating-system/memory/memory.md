---
title: "Memory Management (Hub)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T11:30:00+09:00
tags:
  - operating-system
  - memory
  - hub
---

# Memory Management (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | hub + 9 개 세부 노트 |

**[[../operating-system|↑ OS hub]]**

---

## 1. 한 줄

**가상 메모리** 로 각 프로세스가 자기만의 큰 주소 공간 환상을 가지고,
**MMU + 페이지 테이블** 로 실제 물리 메모리에 매핑한다.

---

## 2. 메모리 계층

```
Register     <  1 ns
L1 cache     ~  1 ns
L2 cache     ~  4 ns
L3 cache     ~ 10 ns
RAM          ~ 100 ns
NVMe SSD     ~ 100 μs
HDD          ~ 10 ms
Network      ~ 100 ms
```

→ "**메모리 계층 차이는 5-6 자릿수**" — 알고리즘 / 자료구조 선택의 큰 이유.

---

## 3. 가상 메모리

[[virtual-memory|↗ Virtual Memory]] — 각 프로세스의 0 ~ 2^48 가상 주소 공간.

---

## 4. 페이지 테이블 & TLB

- [[page-table|↗ Page Table]] — 4-level (x86-64), Huge Page
- [[tlb|↗ TLB]] — Translation Lookaside Buffer

---

## 5. 페이지 폴트 & 교체

- [[page-fault|↗ Page Fault]] — major / minor / COW
- [[page-replacement|↗ Page Replacement]] — LRU / Clock / 2Q / ARC

---

## 6. 할당자

[[allocators|↗ Allocators]] — Buddy / Slab / jemalloc / tcmalloc

---

## 7. mmap

[[mmap|↗ mmap]] — 파일 / 익명 / shared / private

---

## 8. Huge Pages

[[huge-pages|↗ Huge Pages]] — 2MB / 1GB

---

## 9. NUMA

[[numa|↗ NUMA]] — Non-Uniform Memory Access

---

## 10. Swap / OOM

[[swap-oom|↗ Swap / OOM Killer]] — disk swap + OOM 정책

---

## 11. /proc/$PID/maps / smaps

```
55b1c3000000-55b1c3001000 r-xp 00000000 fd:01 1234 /bin/cat
... [heap]
... [stack]
... [vdso]
...
```

가상 주소 공간의 모든 VMA (Virtual Memory Area).

`smaps` 는 RSS / PSS / shared 자세히.

---

## 12. 보는 도구

```bash
free -h                          # 시스템 전체
cat /proc/meminfo
cat /proc/$PID/status            # VmRSS, VmSize
cat /proc/$PID/maps / smaps
pmap -x $PID

vmstat 1                          # si/so (swap), free, cached
top / htop / btop

# 상세
slabtop                           # 커널 slab
numastat -m
numactl --hardware
```

---

## 13. 핵심 키워드

| 키워드 | 의미 |
| --- | --- |
| MMU | 가상→물리 변환 하드웨어 |
| TLB | 변환 캐시 |
| Page (4KB) | 메모리 단위 |
| Page Fault | 페이지 미적재 |
| RSS | Resident Set Size (실제 RAM) |
| VSZ | Virtual Size |
| PSS | Proportional Set Size (공유 분배) |
| Cache (page cache) | 파일 데이터 캐시 |
| Buffer | block device 메타 |
| Slab | 커널 객체 풀 |
| OOM | Out Of Memory |

---

## 14. 면접 핵심

1. **가상 메모리** — 왜 / 어떻게.
2. **페이지 테이블** + **TLB** + **Huge Page**.
3. **mmap vs read/write**.
4. **Copy-on-Write** (fork).
5. **Page replacement** (LRU 근사).
6. **NUMA** — locality.
7. **Heap vs Stack**.
8. **malloc 의 내부** — sbrk / mmap.
9. **메모리 leak vs invalidation**.
10. **OOM Killer** 와 회피.

---

## 15. 학습 자료

- **OSTEP** Ch. 13-23 (메모리 가장 큰 비중)
- **What Every Programmer Should Know About Memory** — Ulrich Drepper
- **The Linux Programming Interface** Ch. 6-7, 49
- **Understanding the Linux Virtual Memory Manager** — Gorman

---

## 16. 관련

- [[../process/pcb]] — mm_struct
- [[../filesystem/filesystem]] — page cache
- [[../io/io]] — mmap 와 I/O
- [[../operating-system|↑ OS hub]]
