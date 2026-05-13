---
title: "Shared Memory — POSIX shm / SysV / mmap"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T14:20:00+09:00
tags:
  - operating-system
  - ipc
  - shm
---

# Shared Memory

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | POSIX shm + SysV + mmap |

**[[ipc|↑ IPC hub]]**

---

## 1. 한 줄

여러 프로세스가 **같은 물리 메모리 페이지** 를 자기 가상 주소에 매핑. **가장 빠른 IPC** (copy X) — 단, 동기화는 별도.

---

## 2. 3 가지 방법

| 방법 | API | 특징 |
| --- | --- | --- |
| **POSIX shm** | `shm_open` + `mmap` | 권장 — file 처럼 |
| **SysV shm** | `shmget` / `shmat` | 옛, 호환성 |
| **mmap MAP_SHARED + file** | `mmap` | 파일도 backing |
| **memfd_create** | `memfd_create` + `mmap` | 익명 + FD 전달 (Linux) |

---

## 3. POSIX shm (권장)

```c
#include <sys/mman.h>
#include <fcntl.h>

// 생성 / 열기
int fd = shm_open("/myshm", O_CREAT | O_RDWR, 0644);
ftruncate(fd, sz);

// 매핑
char *p = mmap(NULL, sz, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
// 데이터 쓰기 / 읽기
p[0] = 42;

// 끝
munmap(p, sz);
close(fd);

// 마지막에 — 다른 프로세스도 모두 unlink 후 사라짐
shm_unlink("/myshm");
```

- `/dev/shm/myshm` 에 생성 (tmpfs)
- 이름은 `/` 로 시작 권장
- 파일 권한과 같음
- 모든 reference 닫히면 해제 (`shm_unlink` 후)

---

## 4. SysV shm (옛)

```c
#include <sys/shm.h>
#include <sys/ipc.h>

key_t key = ftok("/path", 1);
int shmid = shmget(key, sz, IPC_CREAT | 0644);
char *p = shmat(shmid, NULL, 0);

p[0] = 42;

shmdt(p);
shmctl(shmid, IPC_RMID, NULL);
```

- key (정수) 로 식별 — `ftok` 으로 생성
- `ipcs / ipcrm` 도구
- 권한 모델 복잡
- 신규는 POSIX shm 권장

---

## 5. mmap + 파일

```c
int fd = open("/data/buffer.bin", O_RDWR | O_CREAT, 0644);
ftruncate(fd, sz);
char *p = mmap(NULL, sz, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
```

- backing file 있음 → 디스크에 저장
- 응용 재시작해도 데이터 유지
- LMDB, BoltDB 등이 사용

자세히 → [[../memory/mmap]]

---

## 6. memfd_create (Linux 3.17+)

```c
int fd = memfd_create("myshm", MFD_CLOEXEC);
ftruncate(fd, sz);
char *p = mmap(NULL, sz, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);

// 다른 process 에 FD 전달 (Unix socket SCM_RIGHTS)
```

- 익명 (이름 없음)
- 파일 시스템 흔적 X
- FD 전달로 공유
- sealing (immutable) 가능
- IPC 의 모던 표준

---

## 7. fork 후 자동 공유 — anonymous shm

```c
char *p = mmap(NULL, sz, PROT_READ | PROT_WRITE,
               MAP_SHARED | MAP_ANONYMOUS, -1, 0);

if (fork() == 0) {
    p[0] = 42;
    _exit(0);
}
wait(NULL);
assert(p[0] == 42);    // ✅
```

- 부모 ↔ 자식
- 무관한 프로세스 X

---

## 8. 동기화 ⚠️

shm 자체는 통신 — race 막는 건 응용:

```c
// shm 안에 mutex 두기
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_setpshared(&attr, PTHREAD_PROCESS_SHARED);

pthread_mutex_t *mtx = (pthread_mutex_t *)p;
pthread_mutex_init(mtx, &attr);

pthread_mutex_lock(mtx);
// critical
pthread_mutex_unlock(mtx);
```

또는:
- POSIX semaphore (sem_open)
- atomic 변수 + memory barrier
- futex 직접

`PROCESS_SHARED` 옵션 필수 (스레드 간 = PROCESS_PRIVATE 기본).

---

## 9. Robust Mutex

프로세스가 mutex 잡고 죽으면 — 다른 프로세스 영원 대기.

```c
pthread_mutexattr_setrobust(&attr, PTHREAD_MUTEX_ROBUST);

int rc = pthread_mutex_lock(mtx);
if (rc == EOWNERDEAD) {
    // 옛 owner 죽음 — 정리
    pthread_mutex_consistent(mtx);
}
```

→ 프로세스 IPC 의 안전한 mutex 패턴.

---

## 10. 크기 / 한계

```bash
# POSIX shm — tmpfs
df -h /dev/shm

# SysV shm
ipcs -lm     # SHMMAX, SHMALL, SHMMIN
sysctl kernel.shmmax
sysctl kernel.shmall
```

`shmmax` = 단일 segment 최대 (보통 무제한 64bit).
`shmall` = 시스템 전체 page 수.

---

## 11. /dev/shm

```bash
ls -l /dev/shm
# srwxrwxrwx  0  ...  myshm
```

POSIX shm 의 backing tmpfs. cleanup 도구.

---

## 12. /proc/$PID/maps 에서

```
7f4a... rw-s 00000 00:14 1234 /dev/shm/myshm
```

`s` = shared.

---

## 13. 운영 사례

- **PostgreSQL** — `shared_buffers` 가 POSIX shm
- **MySQL** — 일부
- **Redis** — fork 시 자식이 부모 shm 사용
- **Chrome / Firefox** — 프로세스 간 graphics buffer (shm + memfd_create)
- **Wayland / X11** — SHM extension
- **DPDK / mmap NIC** — userspace network

---

## 14. 함정

### 14.1 동기화 누락
race. mutex / atomic.

### 14.2 PROCESS_PRIVATE 기본
shared mutex 안 됨. PROCESS_SHARED.

### 14.3 robust 없이 owner 죽음
영원 대기. ROBUST.

### 14.4 shm_unlink 누락
`/dev/shm` 누적. 정기 cleanup.

### 14.5 backed file vs anonymous 혼동
file mmap = 디스크 write. anonymous = RAM only.

### 14.6 큰 shm + fork CoW
fork 자식이 write → CoW page 복사 → 메모리 ↑.

### 14.7 SysV shm permission 모델
key + uid/gid. POSIX shm 이 간단.

### 14.8 ipcs 의 cleanup
`ipcrm -m <id>` — 사라지지 않는 SysV shm.

---

## 15. 학습 자료

- **The Linux Programming Interface** Ch. 48-49, 54
- `man 7 shm_overview`
- **memfd_create** — kernel docs

---

## 16. 관련

- [[../memory/mmap]]
- [[../synchronization/mutex]] — robust mutex
- [[unix-socket]] — FD 전달
- [[ipc]] — IPC hub
