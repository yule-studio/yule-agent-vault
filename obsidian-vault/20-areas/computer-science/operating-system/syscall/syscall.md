---
title: "System Call (Hub)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T13:45:00+09:00
tags:
  - operating-system
  - syscall
  - hub
---

# System Call (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | hub + 3 개 세부 노트 |

**[[../operating-system|↑ OS hub]]**

---

## 1. 한 줄

응용이 **커널 기능을 요청** 하는 표준 인터페이스. user mode → kernel mode 전환.

---

## 2. user vs kernel mode

자세히 → [[user-kernel-mode]]

```
Ring 3 (User)   — 응용 프로그램, 제한된 명령
Ring 0 (Kernel) — 특권 명령 (I/O, MMU 제어, ...)
```

x86 의 4 ring 중 보통 0 + 3 만 사용.

---

## 3. syscall 호출

```
asm:
  syscall (x86-64) / svc (ARM)
  → CPU 가 kernel mode 진입
  → kernel 의 syscall table 에서 함수 호출
  → 결과 반환 + user mode 복귀
```

```c
// glibc wrapper
ssize_t n = read(fd, buf, sz);

// 직접 syscall
ssize_t n = syscall(SYS_read, fd, buf, sz);
```

---

## 4. POSIX 주요 syscall

자세히 → [[posix-calls]]

| 카테고리 | 예 |
| --- | --- |
| Process | fork, execve, wait, exit, kill, getpid |
| File | open, read, write, close, lseek, stat |
| Memory | mmap, munmap, brk, mlock |
| Network | socket, bind, listen, accept, connect, send, recv |
| IPC | pipe, shmget, msgget, semop |
| Signal | signal, sigaction, kill |
| Time | gettimeofday, clock_gettime, nanosleep |
| Thread | clone, futex |
| 기타 | ioctl, fcntl, prctl |

---

## 5. syscall 비용

- 옛 (`int 0x80`): ~1000 cycles
- 모던 (`syscall`): ~100-200 cycles
- Spectre / Meltdown 패치 (KPTI): +50-200 cycles

→ batch / vectored / io_uring 으로 회피.

자세히 → [[user-kernel-mode#5-cost]]

---

## 6. strace — 디버그 필수

자세히 → [[strace]]

```bash
strace ./app
strace -p $PID
strace -e trace=openat,read,write ./app
strace -c ./app                   # summary
strace -tt -T -f ./app             # 타임스탬프 + 시간 + fork
```

---

## 7. vDSO (virtual Dynamic Shared Object)

자주 호출되는 read-only syscall (gettimeofday, clock_gettime, getpid) 을 **user mode 에서 직접** 실행 — kernel 진입 없음.

```bash
cat /proc/$PID/maps | grep vdso
```

→ `gettimeofday` 가 매우 빠른 이유.

---

## 8. ioctl / fcntl / prctl

특수 / 디바이스 / 프로세스 attribute 조작:

```c
ioctl(fd, FIOCLEX);                  // close-on-exec
fcntl(fd, F_SETFL, O_NONBLOCK);
prctl(PR_SET_NAME, "myname");
```

거의 모든 "이상한 것" 의 catch-all.

---

## 9. 비표준 / Linux-specific

- `clone()` — fork 의 일반화 (namespace)
- `epoll_*` — I/O multiplexing
- `inotify_*` — file watch
- `eventfd / timerfd / signalfd / pidfd`
- `io_uring`
- `bpf()` — eBPF
- `seccomp()`

---

## 10. 세부 노트

| 노트 | 영역 |
| --- | --- |
| [[user-kernel-mode]] | mode 전환 / cost |
| [[posix-calls]] | 주요 syscall 카탈로그 |
| [[strace]] | strace 사용 가이드 |

---

## 11. 함정 (요약)

- syscall 폭증 → 성능 ↓. profiling.
- EINTR 처리 — signal 이 syscall 중단
- errno 의 thread-local
- 작은 read/write 의 비효율
- 32-bit time_t (2038)
- ioctl 호환성

---

## 12. 학습 자료

- **The Linux Programming Interface** — Michael Kerrisk (syscall 바이블)
- **man 2 ...** — 모든 syscall man page
- **syscalls.kernelgrok.com** — 표
- **strace** 공식 문서

---

## 13. 관련

- [[../process/fork-exec]] — fork/exec syscall
- [[../io/io-models]] — read/write
- [[../security/seccomp]] (작성 시)
- [[../operating-system|↑ OS hub]]
