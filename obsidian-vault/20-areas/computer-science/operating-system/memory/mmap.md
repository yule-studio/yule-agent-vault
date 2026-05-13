---
title: "mmap — Memory-Mapped Files / Anonymous"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T12:05:00+09:00
tags:
  - operating-system
  - memory
  - mmap
---

# mmap

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | file / anon / shared / private |

**[[memory|↑ Memory hub]]**

---

## 1. 한 줄

파일 또는 익명 메모리를 **가상 주소 공간에 매핑** → 일반 메모리 접근처럼 사용.
파일 = page cache 와 통합 → zero-copy.

---

## 2. API

```c
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
int munmap(void *addr, size_t length);
int mprotect(void *addr, size_t length, int prot);
int msync(void *addr, size_t length, int flags);
int madvise(void *addr, size_t length, int advice);
```

### 2.1 prot

```
PROT_READ
PROT_WRITE
PROT_EXEC
PROT_NONE
```

### 2.2 flags

| Flag | 의미 |
| --- | --- |
| `MAP_PRIVATE` | CoW (한쪽 write 다른쪽 영향 X) |
| `MAP_SHARED` | write 가 다른 mapping / 파일에 보임 |
| `MAP_ANONYMOUS` | 파일 X (zero-fill) |
| `MAP_FIXED` | addr 강제 (위험) |
| `MAP_POPULATE` | 미리 page fault — preload |
| `MAP_LOCKED` | swap-out 방지 (mlock) |
| `MAP_NORESERVE` | swap 예약 X |
| `MAP_HUGETLB` | huge page |
| `MAP_STACK` | stack 영역 |

---

## 3. 4 가지 조합

```
            FILE                  ANONYMOUS
PRIVATE   파일 read-mostly        malloc 큰 size
                                  CoW 후 자기 사본
SHARED    파일 read+write,         프로세스 간 shm
          여러 프로세스 공유        (POSIX shm_open)
```

---

## 4. 파일 매핑

### 4.1 Read-only

```c
int fd = open("data.bin", O_RDONLY);
struct stat st; fstat(fd, &st);
char *p = mmap(NULL, st.st_size, PROT_READ, MAP_PRIVATE, fd, 0);

// read 처럼 사용
char first = p[0];

munmap(p, st.st_size);
close(fd);
```

### 4.2 Write-back

```c
int fd = open("data.bin", O_RDWR);
char *p = mmap(NULL, sz, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);

p[100] = 0xFF;
msync(p, sz, MS_SYNC);    // 즉시 flush
```

`MAP_SHARED` + 변경 = 디스크에 반영 (kernel page cache → fsync 가 진짜 디스크).

---

## 5. 익명 매핑 — malloc 의 큰 size

```c
char *buf = mmap(NULL, 1 GB, PROT_READ|PROT_WRITE,
                 MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
// 가상 1 GB — 첫 접근 시 zero-fill page

munmap(buf, 1 GB);
```

glibc malloc 도 큰 size 면 자동 mmap (128 KB 기본 threshold).

---

## 6. shared memory (프로세스 간)

```c
// 부모 ↔ 자식 (fork 후)
char *shm = mmap(NULL, sz, PROT_READ|PROT_WRITE,
                 MAP_SHARED | MAP_ANONYMOUS, -1, 0);
if (fork() == 0) {
    shm[0] = 42;
    _exit(0);
}
wait(NULL);
assert(shm[0] == 42);
```

### 6.1 POSIX shm (무관 프로세스)

```c
int fd = shm_open("/myshm", O_CREAT|O_RDWR, 0644);
ftruncate(fd, sz);
char *p = mmap(NULL, sz, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
```

`/dev/shm/myshm` 생성. `shm_unlink` 로 정리.

자세히 → [[../ipc/shared-memory]]

---

## 7. madvise

```c
madvise(p, sz, MADV_SEQUENTIAL);     // 순차 read — readahead
madvise(p, sz, MADV_RANDOM);          // 랜덤 — readahead off
madvise(p, sz, MADV_WILLNEED);        // 미리 read
madvise(p, sz, MADV_DONTNEED);        // 폐기 (재 fault 가능)
madvise(p, sz, MADV_HUGEPAGE);        // THP 권장
madvise(p, sz, MADV_NOHUGEPAGE);
madvise(p, sz, MADV_FREE);            // lazy free (write 시 무효, Linux 4.5+)
madvise(p, sz, MADV_DONTFORK);        // fork 시 자식에 매핑 X
```

자세히 → [[page-fault#10-madvise]]

---

## 8. file read vs mmap

### 8.1 read()

```c
char buf[BUF];
ssize_t n = read(fd, buf, BUF);
// 디스크 → kernel page cache → user buffer (2회 복사)
```

### 8.2 mmap()

```c
char *p = mmap(...);
char c = p[i];        // page fault → kernel page cache (1회), user 가 같은 페이지 봄
```

→ mmap = **zero-copy** (사용자/커널 분리 안 함).

### 8.3 언제 무엇

| 상황 | 추천 |
| --- | --- |
| 큰 파일 random | mmap |
| 작은 파일 / streaming | read |
| 다중 프로세스 공유 | mmap MAP_SHARED |
| sequential | read + posix_fadvise |
| 신뢰성 / 명시적 fsync | read+write (mmap fsync 미묘) |

---

## 9. DB / 검색 엔진의 mmap

- **SQLite** — 옵션 `mmap_size`
- **Boltdb / LMDB** — mmap 중심
- **MongoDB** (옛 MMAPv1)
- **Lucene / Elasticsearch** — index 파일 mmap
- **Redis** — RDB 일부

장점: zero-copy + OS page cache 활용.
단점: error handling 미묘 (SIGBUS), 큰 파일에서 random access 시 page fault 폭증.

---

## 10. 메모리 매핑 + Huge Page

```c
mmap(NULL, sz, PROT_READ|PROT_WRITE,
     MAP_PRIVATE|MAP_ANONYMOUS|MAP_HUGETLB, -1, 0);
```

`/sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages` 미리 reserve.

자세히 → [[huge-pages]]

---

## 11. mremap

```c
new = mremap(old, old_size, new_size, MREMAP_MAYMOVE);
```

기존 mapping 의 크기 변경. realloc 의 mmap 버전.

---

## 12. /proc/$PID/maps 의 mmap

```
7f4a... r--p 00000 fd:01 12345 /data.bin
       └─ PRIVATE / mapped file
       
7f4b... rw-s 00000 fd:01 23456 /dev/shm/myshm
       └─ SHARED / 익명 shm
```

자세히 → [[virtual-memory#4-procpidmaps]]

---

## 13. 함정

### 13.1 vm.max_map_count 한계
기본 65530. ES / Elastic 등은 262144 권장. `sysctl vm.max_map_count`.

### 13.2 SIGBUS
mmap 파일이 truncate / 디스크 풀 → 접근 시 SIGBUS.

### 13.3 비-cooperative concurrency
MAP_SHARED + 여러 프로세스 write — atomic X. lock / fcntl.

### 13.4 munmap 후 access
SIGSEGV.

### 13.5 MAP_FIXED
강제 위치 → 기존 mapping overwrite. 위험.

### 13.6 lazy fault 의 latency
큰 mmap 의 첫 접근 시 page fault 폭증. MAP_POPULATE.

### 13.7 fsync 누락
MAP_SHARED 의 write 가 cache 만 — `msync(MS_SYNC)` + `fsync(fd)`.

### 13.8 32-bit + 큰 mmap
가상 주소 부족. 64-bit 필수.

---

## 14. 학습 자료

- **The Linux Programming Interface** Ch. 49
- `man 2 mmap`, `man 2 madvise`
- **LMDB design** — Howard Chu

---

## 15. 관련

- [[page-fault]]
- [[page-table]]
- [[virtual-memory]]
- [[../ipc/shared-memory]]
- [[memory]] — Memory hub
