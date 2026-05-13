---
title: "Virtual Memory — 가상 메모리"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T11:35:00+09:00
tags:
  - operating-system
  - memory
  - virtual-memory
---

# Virtual Memory

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 가상 주소 공간 + VMA |

**[[memory|↑ Memory hub]]**

---

## 1. 왜

직접 물리 주소 사용 시:
- 프로세스가 다른 프로세스 메모리 침범 가능
- RAM 보다 큰 프로그램 못 실행
- 메모리 단편화 / 재배치 어려움

가상 메모리:
- **격리** — 각 프로세스 자기 공간
- **공유** — read-only 코드 / lib 공유
- **swap** — RAM 보다 큰 작업
- **lazy load** — 필요할 때만 적재

---

## 2. 가상 주소 공간 (x86-64)

```
0xffffffffffffffff
   ...
   Kernel space (커널, 모든 프로세스 공유)
0xffff800000000000
~~~~~~~~~~~~~~~~~~~~~~ (canonical gap)
0x00007fffffffffff
   Stack (↓ 자람)
   ...
   mmap area (lib, anonymous)
   ...
   Heap (↑ 자람)
   BSS / Data
   Text (code)
0x0000000000400000   (일반적 시작)
0x0000000000000000
```

- 총 가상 주소 = 2^48 (canonical) 또는 2^57 (5-level)
- 한 프로세스의 가상 공간 ≈ 128 TB ~ 64 PB

---

## 3. VMA (Virtual Memory Area)

가상 주소 공간을 region 으로 나눔. 각 region 의 권한 / 매핑 종류.

```c
struct vm_area_struct {
    unsigned long vm_start, vm_end;
    pgprot_t      vm_page_prot;     // R/W/X/U
    unsigned long vm_flags;          // VM_READ | VM_WRITE | ...
    struct file  *vm_file;
    unsigned long vm_pgoff;
    ...
};
```

mm_struct 안의 RB tree + list 로 관리.

---

## 4. /proc/$PID/maps

```
address           perms   offset   dev   inode    pathname
55b1...-55b2...   r-xp    00000    fd:01 12345    /bin/cat
55b2...-55b3...   r--p    00100    fd:01 12345    /bin/cat
55b3...-55b4...   rw-p    00200    fd:01 12345    /bin/cat
55b4...-55b5...   rw-p    00000    00:00 0        [heap]
7f4a...-7f4b...   r-xp    00000    fd:01 23456    /lib/libc.so
...
7ffd...           rw-p                            [stack]
7ffd...           r-xp                            [vdso]
ffffffff...       r-xp                            [vsyscall]
```

각 줄 = 1 VMA. 권한 + 종류 + (선택) 파일.

### 4.1 권한

```
r — read
w — write
x — exec
p — private (CoW)
s — shared
```

---

## 5. /proc/$PID/smaps — 자세히

```
55b1...-55b2... r-xp 00000 fd:01 12345 /bin/cat
Size:                 4 kB
KernelPageSize:       4 kB
MMUPageSize:          4 kB
Rss:                  4 kB    ← 실제 RAM
Pss:                  4 kB    ← Proportional (공유 분배)
Shared_Clean:         0 kB
Shared_Dirty:         0 kB
Private_Clean:        4 kB
Private_Dirty:        0 kB
Referenced:           4 kB
Anonymous:            0 kB
LazyFree:             0 kB
AnonHugePages:        0 kB
```

각 VMA 의 메모리 사용 분해.

---

## 6. VSZ / RSS / PSS

| 지표 | 의미 |
| --- | --- |
| **VSZ** (Virtual Size) | 가상 주소 공간 전체 — 의미 ↓ |
| **RSS** (Resident Set Size) | 실제 RAM 차지 — 공유 페이지 중복 카운트 |
| **PSS** (Proportional Set Size) | 공유 페이지를 사용자 수로 나눔 — 정확한 사용 |
| **USS** (Unique Set Size) | 공유 제외, 그 프로세스만 — leak 검출 |

```bash
ps aux                            # VSZ + RSS
cat /proc/$PID/status | grep Vm
smem                               # USS / PSS / RSS
```

VSZ 큼 ≠ RAM 많이 씀. PSS / USS 가 의미 있음.

---

## 7. 메모리 영역

### 7.1 Text (Code)
- ELF 의 `.text`
- read + execute, **shared** (같은 binary 의 모든 프로세스가 페이지 공유)

### 7.2 Data / BSS
- 전역 / static 변수
- Data = 초기화된, BSS = 0 초기화
- read + write, **private (CoW)**

### 7.3 Heap
- `brk()` / `sbrk()` 으로 증가
- malloc / new 가 사용 (작은 건 brk, 큰 건 mmap)

### 7.4 mmap area
- 동적 라이브러리 매핑
- malloc 의 큰 할당
- 메모리 매핑 파일
- 익명 매핑 (`MAP_ANONYMOUS`)

### 7.5 Stack
- 함수 호출 / 지역 변수
- 자동 자람 (페이지 폴트가 확장 트리거)
- 한계 `ulimit -s` (보통 8 MB)

### 7.6 vDSO / vsyscall
- 빠른 시스템 콜 (gettimeofday 등)
- 커널이 user 공간에 매핑

---

## 8. mm_struct (커널)

```c
struct mm_struct {
    struct vm_area_struct *mmap;      // VMA 리스트
    struct rb_root mm_rb;              // RB tree
    pgd_t *pgd;                        // 페이지 테이블 root
    atomic_t mm_users;                  // mm 사용 스레드 수
    unsigned long start_code, end_code;
    unsigned long start_data, end_data;
    unsigned long start_brk, brk;
    unsigned long start_stack;
    unsigned long total_vm;             // 총 VMA pages
    unsigned long rss_stat[NR_MM_COUNTERS];
    ...
};
```

스레드들이 같은 `mm` 공유. fork 시 CoW 로 새 `mm`.

---

## 9. fork + CoW

```
fork():
  새 task_struct + 새 mm_struct
  새 page table (구조만 복사)
  모든 페이지를 read-only 로 표시
  
write 시 → page fault → 그 페이지만 실제 복사
```

→ fork 자체는 가벼움. 자세히 → [[../process/fork-exec]]

---

## 10. malloc 의 구현

```c
void *p = malloc(size);
```

내부:
- 작은 size → arena 의 brk 늘림 (sbrk)
- 큰 size (> 128KB 보통) → `mmap(MAP_ANONYMOUS)` — 별도 VMA
- free 시 brk 의 경우 free list 에 보관 (재사용), mmap 의 경우 munmap

```bash
# malloc 의 페이지 보기
echo 0 > /proc/sys/vm/overcommit_memory     # strict
```

자세히 → [[allocators]]

---

## 11. Overcommit

```bash
cat /proc/sys/vm/overcommit_memory
# 0 = heuristic (기본)
# 1 = 무한 (BGSAVE fork 안전)
# 2 = strict (실제 RAM 안에서만)
```

Linux 는 기본적으로 **overcommit** — malloc 이 성공해도 실제 RAM 부족 시 OOM Killer.

자세히 → [[swap-oom]]

---

## 12. 보안

### 12.1 ASLR (Address Space Layout Randomization)
실행마다 stack / heap / lib 주소가 랜덤 → 공격 어려움.

```bash
cat /proc/sys/kernel/randomize_va_space
# 0 off, 1 mid, 2 full (기본)
```

### 12.2 NX (No-eXecute)
데이터 영역 실행 금지. ELF 의 GNU_STACK 헤더.

### 12.3 W^X
페이지가 W or X — 둘 다 X. JIT 컴파일러는 mmap PROT_WRITE → mprotect PROT_EXEC.

### 12.4 KASLR
커널 주소도 랜덤. Spectre 후 더 중요.

---

## 13. /proc/meminfo

```
MemTotal:       16252428 kB
MemFree:         8123456 kB
MemAvailable:   12345678 kB
Buffers:          234567 kB     # block device 메타
Cached:          3456789 kB     # 파일 page cache
SwapTotal:       8388604 kB
SwapFree:        8388604 kB
Active:          2345678 kB
Inactive:        1234567 kB
Dirty:               123 kB     # 디스크 쓰기 대기
Writeback:             0 kB
AnonPages:       1234567 kB
Mapped:           234567 kB
Shmem:             12345 kB
KReclaimable:     123456 kB
Slab:             345678 kB
HugePages_Total:       0
HugePages_Free:        0
```

자세히 → 각 값의 의미는 `man 5 proc`.

---

## 14. 함정

### 14.1 RSS 만 보고 판단
공유 페이지 중복 카운트. PSS 권장.

### 14.2 VSZ 가 크다고 RAM 많이 씀
가상 — 실제 안 씀.

### 14.3 free 의 "used"
page cache 포함. **`MemAvailable`** 이 진짜.

### 14.4 OOM 발생 후 swap 없으면
즉시 OOM Killer.

### 14.5 ASLR off
디버그 외 절대 X.

### 14.6 mmap 큰 영역 미사용
가상만 — 실제 사용 안 함. 그래도 VMA limit 주의 (`vm.max_map_count`).

### 14.7 Stack overflow
`ulimit -s` 늘리거나 heap 사용.

---

## 15. 학습 자료

- **OSTEP** Ch. 13-18
- **The Linux Programming Interface** Ch. 6, 49
- **What Every Programmer Should Know About Memory** — Drepper

---

## 16. 관련

- [[page-table]]
- [[mmap]]
- [[swap-oom]]
- [[memory]] — Memory hub
