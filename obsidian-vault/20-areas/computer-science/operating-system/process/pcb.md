---
title: "PCB / task_struct — 프로세스 제어 블록"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T09:10:00+09:00
tags:
  - operating-system
  - process
  - pcb
  - linux
---

# PCB / task_struct

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | PCB 개념 + Linux task_struct |

**[[process|↑ Process hub]]**

---

## 1. 한 줄

PCB = OS 가 한 프로세스를 추적하기 위한 **모든 메타데이터** 의 자료구조.
Linux 에서는 `struct task_struct` (수백 필드).

---

## 2. PCB 가 담는 정보

1. **식별** — PID, PPID, TGID, UID, GID
2. **상태** — R / S / D / T / Z
3. **CPU 컨텍스트** — register, PC, SP, fpregs
4. **스케줄링** — priority, nice, vruntime, sched_class
5. **메모리** — mm_struct (page table 루트, VMA)
6. **파일** — files_struct (FD table)
7. **시그널** — signal handlers, mask, pending
8. **IPC** — pipe / shm 핸들
9. **자식 / 부모** — children, parent, real_parent
10. **시간 / 자원 사용** — start_time, utime, stime, rss

---

## 3. Linux `task_struct` — 주요 필드

```c
struct task_struct {
    unsigned int    __state;      // R/S/D/T/Z
    int             prio, static_prio, normal_prio;
    unsigned int    rt_priority;
    const struct sched_class *sched_class;
    struct sched_entity     se;   // CFS 의 vruntime 등
    struct sched_rt_entity  rt;
    struct sched_dl_entity  dl;

    pid_t           pid;          // thread id
    pid_t           tgid;         // process id (스레드 그룹)
    struct task_struct *parent;
    struct task_struct *real_parent;
    struct list_head children;

    struct mm_struct  *mm;        // 메모리 맵 (스레드 공유)
    struct files_struct *files;   // FD table
    struct fs_struct *fs;
    struct signal_struct *signal;
    struct sighand_struct *sighand;
    struct nsproxy *nsproxy;      // namespace
    struct cred *cred;
    ...
};
```

→ 한 task_struct 인스턴스 ≈ 1.7 KB+.

---

## 4. mm_struct — 메모리

```c
struct mm_struct {
    struct vm_area_struct *mmap;   // 또는 RB tree
    struct rb_root mm_rb;
    pgd_t *pgd;                    // 페이지 테이블 루트
    unsigned long start_code, end_code;
    unsigned long start_data, end_data;
    unsigned long start_brk, brk;  // heap
    unsigned long start_stack;
    atomic_t mm_users;             // mm 사용 스레드 수
    ...
};
```

스레드들이 `mm` 을 공유. 프로세스 = 다른 `mm`.

자세히 → [[../memory/virtual-memory]]

---

## 5. files_struct — FD Table

```c
struct files_struct {
    struct fdtable __rcu *fdt;
    spinlock_t file_lock;
    ...
};

struct fdtable {
    unsigned int max_fds;
    struct file __rcu **fd;        // FD 번호 → file 포인터
    unsigned long *close_on_exec;
    unsigned long *open_fds;
};
```

`fd[0]` = stdin, `fd[1]` = stdout, `fd[2]` = stderr.

```bash
ls -l /proc/$PID/fd/
```

---

## 6. signal_struct — 시그널

```c
struct signal_struct {
    struct sigpending shared_pending;
    int group_exit_code;
    struct rlimit rlim[RLIM_NLIMITS];
    ...
};
```

자세히 → [[signals]]

---

## 7. /proc 으로 보기

Linux 는 PCB 의 상당 부분을 `/proc/<pid>/` 로 노출:

```bash
ls /proc/$$/
# cmdline   현재 command
# status    상태 / Uid / VmRSS 등 가독성
# stat      raw 수치 (top 이 파싱)
# maps      메모리 맵 (VMA)
# smaps     상세 메모리
# fd/       열린 FD
# fdinfo/
# environ   환경변수
# cwd ->    현재 디렉터리 (symlink)
# exe ->    실행 파일
# task/     스레드 (각 스레드의 task_struct)
# ns/       namespace
# limits    rlimit
# sched     스케줄 상태
# stack     커널 스택 (D state 디버그)
# cgroup    cgroup 소속
```

### 7.1 status 예

```bash
$ cat /proc/$$/status | head -15
Name:   bash
Umask:  0022
State:  S (sleeping)
Tgid:   12345
Pid:    12345
PPid:   1234
TracerPid: 0
Uid:    1000  1000  1000  1000
Gid:    1000  1000  1000  1000
FDSize: 256
Groups: 4 24 27 1000
VmPeak:  21000 kB
VmSize:  21000 kB
VmRSS:    4500 kB
```

### 7.2 maps 예

```
55b1c3000000-55b1c3001000 r-xp 00000000 fd:01 12345 /bin/cat
55b1c3001000-55b1c3002000 r--p 00001000 fd:01 12345 /bin/cat
7f4a7b800000-7f4a7b821000 r-xp 00000000 fd:01 23456 /lib/x86_64-linux-gnu/libc.so.6
7ffd5a000000-7ffd5a021000 rw-p 00000000 00:00 0     [stack]
```

자세히 → [[../memory/virtual-memory]]

---

## 8. 컨텍스트 스위치 비용

```
1. 현재 task 의 CPU register / PC → task_struct 저장
2. 다음 task 의 register 복원
3. mm 다르면 → CR3 교체 → TLB flush
4. 새 task 실행 시작 → 캐시 cold
```

비용:
- 직접: ~1-10 μs
- 간접: TLB / L1-L3 캐시 miss → 수십 μs+

같은 프로세스 안의 스레드 스위치 = mm 같음 → TLB flush X → 가벼움.

자세히 → [[../scheduling/context-switch]]

---

## 9. namespace (컨테이너)

PCB 의 `nsproxy` 가 namespace 들의 포인터:
- `pid_ns` — PID 격리
- `mnt_ns` — 마운트
- `net_ns` — 네트워크
- `uts_ns` — hostname
- `ipc_ns`
- `user_ns`
- `cgroup_ns`

자세히 → [[../virtualization/namespace]]

---

## 10. cred — 권한

```c
struct cred {
    kuid_t uid, gid, euid, egid, suid, sgid, fsuid, fsgid;
    kernel_cap_t cap_effective, cap_permitted, cap_inheritable;
    ...
};
```

자세히 → [[../security/permissions]]

---

## 11. PID vs TID vs TGID

| 용어 | 의미 |
| --- | --- |
| **PID** (커널 안) | task (= 스레드) 번호 |
| **TGID** | thread group ID = 사용자가 보는 "PID" |
| **TID** | thread ID (= 커널의 PID) |

```c
getpid()     → tgid    // 사용자 의도의 PID
gettid()     → pid     // 스레드 자신의 ID
```

`ps -L` 로 스레드별 보기.

---

## 12. 함정

### 함정 1 — task_struct 의 메모리 부담
프로세스 / 스레드 100만 = ~ 2 GB.

### 함정 2 — /proc 의 race
`/proc/<pid>` 읽는 사이 프로세스 죽으면 ENOENT.

### 함정 3 — `ps` 의 비싼 호출
대량 호출 시 매번 /proc 스캔. 모니터링은 sampling.

### 함정 4 — `getpid()` vs `gettid()`
멀티스레드 디버그 시 혼동.

### 함정 5 — fork 후 mm 공유 환상
fork = CoW 로 공유 (lazy). pthread = 진짜 공유.

---

## 13. 학습 자료

- **Linux Kernel Development** Ch. 3 (Process Management)
- **The Linux Programming Interface** Ch. 24-26
- **Understanding the Linux Kernel** Ch. 3
- `Documentation/filesystems/proc.rst`

---

## 14. 관련

- [[fork-exec]] — task_struct 생성
- [[signals]]
- [[../scheduling/context-switch]]
- [[process]] — Process hub
