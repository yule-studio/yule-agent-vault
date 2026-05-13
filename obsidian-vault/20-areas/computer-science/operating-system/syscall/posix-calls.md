---
title: "POSIX 시스템 콜 카탈로그"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T13:55:00+09:00
tags:
  - operating-system
  - syscall
  - posix
---

# POSIX 시스템 콜 카탈로그

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 주요 syscall 분류 |

**[[syscall|↑ Syscall hub]]**

---

## 1. 분류별 핵심 syscall

---

## 2. 프로세스 관리

| Syscall | 의미 |
| --- | --- |
| `fork()` | 자식 프로세스 복제 |
| `clone()` | fork 의 일반화 (Linux) |
| `vfork()` | fork (옛, 비추) |
| `execve()` | 새 프로그램으로 교체 |
| `wait4 / waitpid` | 자식 종료 대기 |
| `exit / _exit / exit_group` | 종료 |
| `kill / tkill / tgkill` | 시그널 전송 |
| `getpid / getppid / gettid` | PID |
| `setpgid / getpgid` | 프로세스 그룹 |
| `setsid` | 새 session |

자세히 → [[../process/fork-exec]], [[../process/signals]]

---

## 3. 파일 I/O

| Syscall | 의미 |
| --- | --- |
| `open / openat` | 파일 열기 |
| `creat` | open + O_CREAT|O_WRONLY|O_TRUNC |
| `read / readv / pread / preadv` | 읽기 |
| `write / writev / pwrite / pwritev` | 쓰기 |
| `close` | 닫기 |
| `lseek` | 오프셋 변경 |
| `dup / dup2 / dup3` | FD 복제 |
| `fcntl` | FD 속성 (NONBLOCK, CLOEXEC, lock) |
| `ioctl` | 디바이스 / 특수 |
| `pipe / pipe2` | pipe 생성 |
| `mkfifo` | FIFO |
| `sendfile / splice / tee / copy_file_range` | zero-copy |

자세히 → [[../filesystem/inode-dentry]]

---

## 4. 파일 시스템

| Syscall | 의미 |
| --- | --- |
| `stat / lstat / fstat / statx` | 메타 |
| `chmod / fchmod / chown / fchown` | 권한 |
| `link / unlink / rename / symlink / readlink` | 링크 |
| `mkdir / rmdir / getdents` | 디렉토리 |
| `mount / umount` | 마운트 |
| `chroot / pivot_root` | 루트 변경 |
| `chdir / fchdir / getcwd` | cwd |
| `fsync / fdatasync / sync_file_range / sync / syncfs` | durability |
| `truncate / ftruncate` | 크기 |
| `mknod` | 디바이스 노드 |
| `access / faccessat` | 권한 확인 |
| `utime / utimensat` | 타임스탬프 |
| `getxattr / setxattr / listxattr` | xattr |
| `flock / fcntl(F_SETLK)` | 파일 락 |
| `inotify_* / fanotify_*` | 변경 감시 |
| `statfs / fstatfs` | fs 정보 |

---

## 5. 메모리

| Syscall | 의미 |
| --- | --- |
| `mmap / munmap / mremap` | 매핑 |
| `mprotect` | 권한 변경 |
| `madvise` | hint |
| `mlock / mlockall / munlock` | swap 방지 |
| `brk / sbrk` | heap |
| `msync` | mmap flush |
| `mincore` | 페이지 RAM 에 있나 |
| `process_vm_readv / writev` | cross-process memory |

자세히 → [[../memory/mmap]]

---

## 6. 네트워크 (BSD socket)

| Syscall | 의미 |
| --- | --- |
| `socket` | 생성 |
| `bind` | local 주소 |
| `listen` | listen |
| `accept / accept4` | 연결 수락 |
| `connect` | 연결 |
| `send / sendto / sendmsg` | 송신 |
| `recv / recvfrom / recvmsg` | 수신 |
| `shutdown` | 한쪽 닫기 |
| `setsockopt / getsockopt` | 옵션 |
| `getsockname / getpeername` | 주소 |
| `socketpair` | 한 쌍 |

옵션 예: SO_REUSEADDR, SO_REUSEPORT, TCP_NODELAY, SO_KEEPALIVE.

---

## 7. I/O Multiplexing

| Syscall | 의미 |
| --- | --- |
| `select / pselect` | 옛 |
| `poll / ppoll` | 일반 |
| `epoll_create1 / epoll_ctl / epoll_wait` | Linux |
| `kqueue / kevent` | BSD |
| `io_uring_setup / enter / register` | Linux 5.1+ |

자세히 → [[../io/select-poll-epoll]], [[../io/io-uring]]

---

## 8. 시그널

| Syscall | 의미 |
| --- | --- |
| `signal` (deprecated) / `sigaction` | 핸들러 |
| `kill / tkill / tgkill / sigqueue` | 송신 |
| `pause / sigsuspend` | 대기 |
| `sigwait / sigwaitinfo / sigtimedwait` | 동기 수신 |
| `signalfd` | FD 로 |
| `sigaltstack` | 핸들러용 별도 stack |
| `sigprocmask / pthread_sigmask` | mask |
| `sigreturn` (자동) | handler 복귀 |
| `prctl(PR_SET_PDEATHSIG)` | 부모 죽으면 시그널 |

자세히 → [[../process/signals]]

---

## 9. IPC

| Syscall | 의미 |
| --- | --- |
| `pipe / pipe2` | 익명 pipe |
| `mkfifo / mknod` | 명명 FIFO |
| `socketpair` | 양방향 |
| `shmget / shmat / shmdt / shmctl` | System V shm |
| `shm_open / shm_unlink` | POSIX shm |
| `mmap MAP_SHARED` | shm |
| `msgget / msgsnd / msgrcv / msgctl` | System V message queue |
| `mq_open / mq_send / mq_receive` | POSIX message queue |
| `semget / semop / semctl` | System V semaphore |
| `sem_open / sem_wait / sem_post / sem_close / sem_unlink` | POSIX semaphore |
| `eventfd / timerfd / signalfd / pidfd` | event FD |

자세히 → [[../ipc/ipc]]

---

## 10. 스레드

| Syscall | 의미 |
| --- | --- |
| `clone` | task 생성 (스레드 = mm 공유 task) |
| `futex` | mutex / condvar / semaphore 의 기반 |
| `set_robust_list / get_robust_list` | robust mutex |
| `tgkill` | 특정 스레드 시그널 |
| `sched_setaffinity / getaffinity` | CPU affinity |
| `sched_setscheduler / getscheduler` | 정책 |
| `sched_setattr / getattr` | DEADLINE 등 |
| `sched_yield` | 양보 |

자세히 → [[../threads/threads]]

---

## 11. 시간

| Syscall | 의미 |
| --- | --- |
| `gettimeofday` | sec + usec (옛) |
| `clock_gettime / clock_getres` | nsec (CLOCK_REALTIME, CLOCK_MONOTONIC, ...) |
| `clock_nanosleep / nanosleep / sleep` | sleep |
| `alarm` | SIGALRM (옛) |
| `setitimer / getitimer` | 주기적 |
| `timer_create / timer_settime` | POSIX timer |
| `timerfd_create / timerfd_settime` | FD 로 |
| `adjtimex` | NTP |

| Clock | 의미 |
| --- | --- |
| `CLOCK_REALTIME` | wall clock — NTP 로 변함 |
| `CLOCK_MONOTONIC` | 단조 — sleep 빼고 |
| `CLOCK_BOOTTIME` | sleep 포함 |
| `CLOCK_THREAD_CPUTIME_ID` | thread CPU |
| `CLOCK_PROCESS_CPUTIME_ID` | process CPU |

→ duration / timeout 은 MONOTONIC.

---

## 12. 권한 / 사용자

| Syscall | 의미 |
| --- | --- |
| `getuid / setuid / seteuid / setresuid` | UID |
| `getgid / setgid / setresgid / setgroups` | GID |
| `capset / capget` | capability |
| `prctl` | per-process flags |
| `seccomp / prctl(PR_SET_SECCOMP)` | seccomp filter |
| `ptrace` | debugger |
| `getrlimit / setrlimit / prlimit` | rlimit |

자세히 → [[../security/permissions]]

---

## 13. namespace / 가상화

| Syscall | 의미 |
| --- | --- |
| `unshare` | namespace 분리 |
| `setns` | join namespace |
| `clone(CLONE_NEW*)` | 새 namespace |
| `mount` | per-namespace mount |

자세히 → [[../virtualization/namespace]]

---

## 14. eBPF / Tracing

| Syscall | 의미 |
| --- | --- |
| `bpf()` | eBPF 프로그램 로드 / map |
| `perf_event_open` | perf counter |
| `ptrace` | 디버거 |
| `process_vm_readv / writev` | 다른 process 메모리 |

---

## 15. 자주 잘못 쓰는 패턴

### 15.1 errno not thread-local 가정
glibc 의 `errno` 는 TLS — thread-safe.

### 15.2 partial read/write
```c
ssize_t total = 0;
while (total < n) {
    ssize_t r = read(fd, buf + total, n - total);
    if (r < 0) {
        if (errno == EINTR) continue;
        return -1;
    }
    if (r == 0) break;     // EOF
    total += r;
}
```

### 15.3 close 후 error
일부 OS / NFS = close 가 진짜 fsync. close 에러 무시 위험.

### 15.4 select / poll / epoll 의 timeout 단위
ms (epoll), tv (select), ms (poll). 통일 X.

### 15.5 syscall vs glibc wrapper
glibc 가 errno / signal 처리. 직접 `syscall()` 사용 시 주의.

---

## 16. 학습 자료

- **The Linux Programming Interface** (Kerrisk)
- **man 2 ...** — 모든 syscall
- **man 7 ...** — 개념 (epoll, pipe, signal-safety)
- **syscalls.kernelgrok.com**

---

## 17. 관련

- [[strace]]
- [[user-kernel-mode]]
- [[syscall]] — Syscall hub
