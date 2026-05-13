---
title: "Page Fault — 페이지 폴트"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T11:50:00+09:00
tags:
  - operating-system
  - memory
  - page-fault
---

# Page Fault

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | minor / major / segfault / COW |

**[[memory|↑ Memory hub]]**

</br>

**[[memory|↑ Memory hub]]**

---

## 1. 한 줄

CPU 가 가상 주소 접근 → PTE 가 P=0 또는 잘못된 권한 → **trap to kernel** → 커널이 해결.

---

## 2. 종류

| 종류 | 의미 |
| --- | --- |
| **Minor (soft) fault** | 페이지 메타데이터 만들기 / CoW / shared page mapping (디스크 X) |
| **Major (hard) fault** | 디스크에서 swap-in / 파일 read (디스크 I/O O) |
| **Invalid (segfault)** | 잘못된 접근 → SIGSEGV |

```bash
ps -o pid,min_flt,maj_flt,comm
/proc/$PID/stat        # min_flt, maj_flt
```

---

## 3. Minor Fault 예

### 3.1 첫 접근 (Demand Paging)

```c
char *p = mmap(NULL, 1 GB, PROT_READ|PROT_WRITE, MAP_ANON|MAP_PRIVATE, -1, 0);
// 아직 페이지 X — 가상만
p[0] = 1;     // minor fault — 0 채운 페이지 할당
```

→ malloc 큰 size 도 같은 패턴.

### 3.2 Copy-on-Write

```c
fork();
// 부모/자식 모두 같은 페이지 read
parent: write       // → minor fault → 페이지 복사
```

### 3.3 Shared Library 첫 호출

dlopen 후 첫 함수 호출 → 페이지 폴트 → ELF 의 해당 페이지 매핑.

---

## 4. Major Fault 예

### 4.1 Swap-in

```
ps 가 sleep 오래 → swap 으로 떨어짐
재실행 시 → major fault → 디스크 read
```

### 4.2 mmap 파일 read

```c
char *p = mmap(NULL, sz, PROT_READ, MAP_PRIVATE, fd, 0);
// 페이지 X
char c = p[1024];   // major fault — 페이지 cache 에 없으면 디스크 read
```

OS 의 page cache 에 있으면 minor (디스크 X).

---

## 5. 핸들링

```
1. CPU trap (#PF)
2. CR2 = 폴트 발생 가상 주소
3. 커널 page_fault_handler:
   - VMA 찾기
   - 권한 검사
   - 권한 OK → 페이지 준비 (alloc, swap-in, file read)
   - 권한 NX → SIGSEGV
4. PTE 업데이트
5. 사용자 코드 재실행
```

---

## 6. SIGSEGV 케이스

- NULL deref: `*(int *)0 = 1`
- stack overflow: 무한 재귀
- write to read-only: `char *p = "literal"; p[0]='x';`
- exec on NX page

```bash
# core dump
ulimit -c unlimited
./bad
gdb ./bad core
```

자세히 → [[../process/signals#9-sigsegv--sigbus--메모리-오류]]

---

## 7. Lazy Page Allocation 의 의미

```c
char *p = malloc(1 GB);    // 거의 즉시 — 가상만
memset(p, 0, 1 GB);         // 여기서 실제 1 GB page 할당 (수많은 minor fault)
```

→ "메모리 할당이 빠른 듯" 보이지만 첫 접근 시 비용.

---

## 8. mlock / mlockall — 강제 RAM

```c
mlock(p, sz);
mlockall(MCL_CURRENT | MCL_FUTURE);
```

- swap-out 방지
- 페이지 미리 채움
- RT / DB / 보안 (메모리 secret) 에 사용

`ulimit -l` 제한.

---

## 9. MAP_POPULATE

```c
mmap(..., MAP_PRIVATE | MAP_POPULATE, ...);
```

→ mmap 호출 시 모든 페이지 prefault. lazy 회피.

---

## 10. madvise

```c
madvise(p, sz, MADV_SEQUENTIAL);    // readahead 최적
madvise(p, sz, MADV_RANDOM);         // readahead 끄기
madvise(p, sz, MADV_WILLNEED);       // 미리 read
madvise(p, sz, MADV_DONTNEED);       // 폐기 (재 fault 가능)
madvise(p, sz, MADV_HUGEPAGE);       // THP 권장
```

OS 에 의도 알림.

---

## 11. Watcher / Profiler

```bash
# 페이지 폴트 통계
sar -B 1                       # pgflt/s pgmajflt/s

# 프로세스별
pidstat -t -r 1                # minflt/s majflt/s

# 시스템 전체
vmstat 1                       # si/so (swap) 보이면 major 많음
```

---

## 12. 함정

### 12.1 큰 메모리 사전 할당의 환상
malloc 큰 size = 가상만. 첫 접근 시 polling fault → latency spike.

### 12.2 RT task + page fault
ms 단위 지연. `mlockall(MCL_CURRENT|MCL_FUTURE)`.

### 12.3 swap on → major fault → tail latency
swap 끄거나 안 떨어지게.

### 12.4 SIGSEGV 무시
defensive 코드 + sanitizer.

### 12.5 lazy zero page
처음엔 모든 page 가 같은 zero page (CoW). write 시 분리.

### 12.6 fork 후 write 폭주
모든 부모 페이지 CoW 복사 → memory ↑. Redis 의 fork issue.

### 12.7 page cache 압박
free memory ↓ → major fault ↑ on read.

---

## 13. 학습 자료

- **OSTEP** Ch. 21-22
- **The Linux Programming Interface** Ch. 49
- **Linux kernel** `mm/memory.c` (handle_pte_fault)

---

## 14. 관련

- [[page-table]]
- [[mmap]]
- [[page-replacement]]
- [[swap-oom]]
- [[memory]] — Memory hub
