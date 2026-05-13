---
title: "Memory Allocators — Buddy / Slab / jemalloc / tcmalloc"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T12:00:00+09:00
tags:
  - operating-system
  - memory
  - allocator
---

# Memory Allocators

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 커널 + 사용자 공간 |

**[[memory|↑ Memory hub]]**

---

## 1. 두 레벨

```
┌─────────────────────────────┐
│  Application (malloc/free)   │
└─────────────┬────────────────┘
              │
┌─────────────▼────────────────┐
│  libc allocator (glibc/jemal) │
│  brk / mmap                   │
└─────────────┬────────────────┘
              │
┌─────────────▼────────────────┐
│  Kernel (Buddy + Slab)        │
└──────────────────────────────┘
```

---

## 2. 커널 — Buddy System

페이지 단위 (4 KB) 할당. 2^k page 의 자유 리스트.

```
order 0: 4 KB
order 1: 8 KB
order 2: 16 KB
...
order 10: 4 MB
```

할당:
- 요청 = order k → free list 에서 2^k pages
- 없으면 더 큰 block 을 split

해제:
- buddy (인접 같은 size) 와 합침 → defragment

```bash
cat /proc/buddyinfo
# Node 0, zone Normal     500 200 100 50 ...
# (order 0, 1, 2 ... 의 free block 수)
```

---

## 3. Slab / SLUB / SLOB (커널)

작은 객체 (task_struct, dentry, inode) 용. Buddy 위의 layer.

```
cache_per_object_type:
  task_struct: 미리 만든 객체 pool
  alloc → 한 객체 반환 (페이지 새로 할당 안 함)
  free → pool 에 반환 (defragment 안 함)
```

- **SLAB** — 옛, complex
- **SLUB** — 현재 default, 단순
- **SLOB** — 작은 시스템

```bash
slabtop
cat /proc/slabinfo
```

내부 단편화 적음. 객체 type 별 cache.

---

## 4. 사용자 — glibc malloc (ptmalloc)

```
small (<= 64 byte) → tcache (per-thread)
small (<= 1024)     → fastbin
medium               → smallbins / largebins
large (> 128 KB 기본) → mmap (별도 VMA)
```

```bash
# 환경변수
MALLOC_ARENA_MAX=2 ./app          # arena 수 제한 (멀티스레드 메모리 ↑)
MALLOC_TRIM_THRESHOLD_=128000      # munmap 임계
mallopt(M_MMAP_THRESHOLD, ...)
mallopt(M_TRIM_THRESHOLD, ...)
```

- 다중 arena (코어 수 × 8) → 락 경합 ↓
- thread cache (per-thread fast path)

---

## 5. jemalloc

Facebook / Mozilla / Rust 가 선호.

특징:
- arena per CPU
- multi-size bin
- 작은 단편화
- 정확한 통계 (`MALLOC_CONF=stats_print:true`)

```bash
LD_PRELOAD=/usr/lib/libjemalloc.so ./app
```

Rust 기본 (linux), Firefox 사용.

---

## 6. tcmalloc (Google)

Google Performance Tools.

특징:
- Thread-cached
- 매우 빠른 작은 alloc
- 페이지 단위 central
- profiler 강력 (`pprof`)

```bash
LD_PRELOAD=/usr/lib/libtcmalloc.so ./app
HEAPPROFILE=/tmp/heap.prof ./app
```

---

## 7. mimalloc (Microsoft)

새 allocator. 매우 빠른 작은 alloc + 좋은 multi-thread.

---

## 8. Arena Allocator

```c
char buffer[10 MB];
char *p = buffer;

void *arena_alloc(size_t n) {
    void *r = p;
    p += n;
    return r;
}

// 한 번에 reset
p = buffer;
```

- 매우 빠름 — bump pointer
- free 개별 X — 통째로 reset
- request 처리 / 컴파일러 / 게임 frame

---

## 9. Stack Allocator (alloca)

```c
char *buf = alloca(1024);    // 스택 위에 — 자동 해제
```

- 매우 빠름
- 스택 한계 (`ulimit -s`, 보통 8 MB)
- 함수 끝나면 자동 해제

가변 길이 array (VLA) 도 비슷.

---

## 10. Object Pool

자주 alloc/free 되는 같은 type 의 객체:

```c
typedef struct { Object *next; } FreeNode;
FreeNode *free_list = NULL;

Object *obj_alloc() {
    if (free_list) {
        FreeNode *n = free_list;
        free_list = n->next;
        return (Object *)n;
    }
    return malloc(sizeof(Object));
}

void obj_free(Object *o) {
    FreeNode *n = (FreeNode *)o;
    n->next = free_list;
    free_list = n;
}
```

→ Slab 의 사용자 공간 버전.

---

## 11. RC / GC vs Manual

| | 수동 (malloc/free) | RC | GC |
| --- | --- | --- | --- |
| Predictable | ✅ | ✅ | ❌ pause |
| 누수 | 위험 | 순환 참조 | 안전 |
| 속도 | 가장 빠름 | 빠름 | 가변 |
| 언어 | C/C++/Rust | Swift, C++ shared_ptr | Java/Go/Python |

---

## 12. 메모리 단편화

### 12.1 외부 단편화
free 블록들이 흩어져 있어 큰 요청 못 함. Buddy 가 줄임.

### 12.2 내부 단편화
요청보다 큰 단위 할당 (예: 1 byte 요청 → 4 KB 페이지). Slab / size class 이슈.

---

## 13. 진단

```bash
# 누수
valgrind --leak-check=full ./app

# heap profile
LD_PRELOAD=libjemalloc.so MALLOC_CONF=prof:true ./app

# Go
go tool pprof http://host:port/debug/pprof/heap

# Java
jcmd $PID GC.heap_dump heap.bin
jhat / Eclipse MAT

# Python
tracemalloc / objgraph
```

---

## 14. 함정

### 14.1 large alloc 의 mmap
> 128 KB (기본) → mmap → munmap on free → 페이지 fault on next.

### 14.2 잦은 small alloc
arena lock 경합. jemalloc / tcmalloc.

### 14.3 fragment 후 RSS 유지
free 했지만 mmap 안 함 → RSS 그대로. `malloc_trim()`.

### 14.4 멀티스레드 + glibc malloc
arena 8 × 코어 가능. `MALLOC_ARENA_MAX=2`.

### 14.5 alloca 큰 size
stack overflow.

### 14.6 GC pause
Java / Go 의 STW. 모니터링.

### 14.7 leak check 안 함
누적 → OOM Killer.

---

## 15. 학습 자료

- **OSTEP** Ch. 17
- **The Slab Allocator** — Bonwick (1994)
- **jemalloc paper** — Evans
- **tcmalloc** — Google blog

---

## 16. 관련

- [[mmap]]
- [[virtual-memory]]
- [[memory]] — Memory hub
