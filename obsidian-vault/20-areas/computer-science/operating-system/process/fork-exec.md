---
title: "fork / exec / wait — 프로세스 생성과 교체"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T09:15:00+09:00
tags:
  - operating-system
  - process
  - fork
  - exec
  - cow
---

# fork / exec / wait

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | UNIX 생성 모델 + CoW |

**[[process|↑ Process hub]]**

---

## 1. UNIX 의 독특한 디자인

다른 OS = "process 생성" 1 호출.
UNIX = **fork (복제) + exec (교체)** 2 단계.

→ 자식의 환경 (FD, env, cwd, signal) 을 정밀 제어 가능.
→ 셸이 redirection / pipe 를 자연스럽게 구현.

---

## 2. fork — 프로세스 복제

```c
#include <unistd.h>
pid_t pid = fork();

if (pid < 0) {
    // 실패
} else if (pid == 0) {
    // child — pid == 0
    printf("child %d\n", getpid());
} else {
    // parent — pid == child pid
    printf("parent: child is %d\n", pid);
}
```

### 2.1 자식이 부모로부터 복제하는 것
- 메모리 (Code/Data/Heap/Stack)
- FD table
- 환경변수, cwd
- signal 핸들러 (대부분)
- UID/GID

### 2.2 자식이 다른 것
- PID (다른 값)
- PPID (부모의 PID)
- fork() 반환값
- 시간 / 자원 사용량 reset
- pending signal reset
- file lock 비복사

---

## 3. Copy-on-Write (CoW)

fork = 메모리 전체 복사가 아니다.

```
부모의 모든 페이지를 자식에 mapping
모두 read-only 로 표시
한쪽이 write → page fault → 그 페이지만 실제 복사
```

→ fork 자체는 **빠르다 + 메모리 적게**.
→ 큰 프로세스도 fork 가능 (페이지 테이블 자체 복제는 있음).

### 3.1 CoW 의 함정

- fork 후 자식이 많이 쓰면 → 결국 메모리 ≈ 2 배
- Redis 의 BGSAVE 시 큰 write 가 일어나면 메모리 폭증

---

## 4. vfork (deprecated)

```c
pid_t pid = vfork();
```

- 자식이 exec 또는 _exit 까지 부모 mm 공유
- 부모는 대기 (자식 끝까지)
- 옛 (CoW 전) 최적화. **이제 사용 X** — CoW 가 충분.

---

## 5. exec — 새 프로그램으로 교체

```c
execl ("/bin/ls", "ls", "-l", NULL);
execlp("ls", "ls", "-l", NULL);          // PATH 검색
execv ("/bin/ls", argv);
execve("/bin/ls", argv, envp);            // 진짜 시스템 콜
```

| 변형 | 의미 |
| --- | --- |
| `l` (list) | 가변 인자 list |
| `v` (vector) | argv 배열 |
| `p` (path) | PATH 환경변수 검색 |
| `e` (env) | env 명시 |

### 5.1 exec 의 동작

```
1. 같은 PID 유지
2. 메모리 통째로 교체 (새 ELF 로드)
3. 새 stack / heap
4. FD 는 유지 (close-on-exec 제외)
5. signal 처리 핸들러 reset → SIG_DFL
6. exec() 성공 시 절대 return X
```

→ "I am still the same process but executing different code".

---

## 6. fork + exec 패턴

```c
pid_t pid = fork();
if (pid == 0) {
    execvp("ls", (char *[]){"ls", "-l", NULL});
    perror("exec");          // exec 실패 시만 도달
    _exit(127);
}
// parent 계속
int status;
waitpid(pid, &status, 0);
```

셸의 표준 패턴.

---

## 7. wait / waitpid

```c
pid_t pid = wait(&status);                  // 아무 자식
pid_t pid = waitpid(-1, &status, 0);        // 동일
pid_t pid = waitpid(child, &status, 0);     // 특정
pid_t pid = waitpid(child, &status, WNOHANG);  // non-blocking
pid_t pid = waitpid(-1, &status, WNOHANG | WUNTRACED);
```

### 7.1 status 매크로

```c
if (WIFEXITED(status))   exit_code = WEXITSTATUS(status);   // 정상 종료
if (WIFSIGNALED(status)) sig       = WTERMSIG(status);       // 시그널
if (WIFSTOPPED(status))  sig       = WSTOPSIG(status);       // SIGSTOP
if (WCOREDUMP(status))   ...
```

### 7.2 wait 안 하면

자식이 Zombie → PID / PCB 누적.
자세히 → [[orphan-zombie]]

---

## 8. exit / _exit / atexit

```c
exit(0);          // atexit / stdio flush / dtor
_exit(0);         // 즉시 시스템 콜 — fork 의 자식 exec 실패 후
```

### 8.1 atexit

```c
atexit(cleanup);
```

main return / exit() 시 호출. _exit() 는 호출 X.

---

## 9. fork 와 멀티스레드의 위험 ⚠️

```
프로세스 안에 N 스레드.
어느 스레드가 fork() 호출.
→ 자식은 호출 스레드만 복제. 다른 스레드 사라짐.
→ 다른 스레드가 잡고 있던 mutex = 영원히 잠긴 상태.
→ 자식이 그 락에 접근 → 영원히 대기.
```

해결:
- fork 직후 **즉시 exec** — 락 사용 안 함
- `pthread_atfork(prepare, parent, child)` — 락 처리
- 가능하면 **fork-only model 회피** (멀티스레드 + fork 안 섞기)

---

## 10. posix_spawn

fork + exec 의 안전한 단축:

```c
#include <spawn.h>
posix_spawn(&pid, "/bin/ls", &file_actions, &attr, argv, envp);
```

스레드 모델 / 락 문제 회피. macOS / 임베디드 권장.

---

## 11. 셸의 redirection 구현

```bash
ls > out.txt
```

```c
pid = fork();
if (pid == 0) {
    int fd = open("out.txt", O_WRONLY|O_CREAT|O_TRUNC, 0644);
    dup2(fd, 1);            // stdout → out.txt
    close(fd);
    execlp("ls", "ls", NULL);
}
waitpid(pid, NULL, 0);
```

`dup2()` 가 핵심 — exec 전에 자식의 FD 재배치.

### 11.1 Pipe

```bash
ls | wc -l
```

```c
int pipefd[2];
pipe(pipefd);

if (fork() == 0) {        // ls
    dup2(pipefd[1], 1);
    close(pipefd[0]); close(pipefd[1]);
    execlp("ls", "ls", NULL);
}
if (fork() == 0) {        // wc
    dup2(pipefd[0], 0);
    close(pipefd[0]); close(pipefd[1]);
    execlp("wc", "wc", "-l", NULL);
}
close(pipefd[0]); close(pipefd[1]);
wait(NULL); wait(NULL);
```

자세히 → [[../ipc/pipe-fifo]]

---

## 12. 환경 변수

```c
extern char **environ;
char *path = getenv("PATH");
setenv("LANG", "ko_KR.UTF-8", 1);
unsetenv("DEBUG");

// exec 에 명시
char *env[] = {"PATH=/bin", "HOME=/tmp", NULL};
execve("/bin/sh", argv, env);
```

`fork` 는 env 복사. `exec` 는 envp 그대로.

---

## 13. /proc 에서 보기

```bash
cat /proc/$PID/cmdline | tr '\0' ' '
cat /proc/$PID/environ | tr '\0' '\n'
ls -l /proc/$PID/cwd
ls -l /proc/$PID/exe
ls -l /proc/$PID/fd/
cat /proc/$PID/status | grep -E 'PPid|TracerPid'
```

---

## 14. 함정

### 함정 1 — fork 후 stdio buffer
```c
printf("hello\n");
fork();
// hello 가 2 번 출력될 수 있음 (line buffered 환경 따라)
fflush(stdout);
fork();
```

### 함정 2 — `exit()` vs `_exit()` in child after fork
exec 실패 후 `exit()` → atexit 가 부모와 충돌. `_exit()` 사용.

### 함정 3 — close-on-exec 누락
자식 exec 후에도 FD 살아 있음 → 누수 / 보안. `O_CLOEXEC` 또는 `fcntl(fd, F_SETFD, FD_CLOEXEC)`.

### 함정 4 — 좀비 + 핸들러
SIGCHLD 핸들러 안에서 `waitpid(-1, &s, WNOHANG)` 로 모두 회수.

### 함정 5 — fork 폭발
재귀 fork (bash fork bomb `:(){ :|:& };:`). `ulimit -u` / cgroup pids.max.

### 함정 6 — exec 후 signal 핸들러
SIG_DFL / SIG_IGN 으로 reset. 사용자 핸들러 사라짐.

### 함정 7 — clone() 의 변형
`clone()` 으로 namespace / FS 등 옵션 가능 — Docker 가 사용.

---

## 15. clone (Linux) — fork 의 일반화

```c
pid_t pid = clone(child_fn, child_stack,
                  CLONE_NEWNS | CLONE_NEWPID | CLONE_NEWNET,
                  arg);
```

옵션 (CLONE_*):
- `CLONE_VM` — mm 공유 (스레드)
- `CLONE_FS / FILES / SIGHAND` — 공유
- `CLONE_NEWNS / NEWPID / NEWNET / ...` — 새 namespace (컨테이너)

→ pthread, container runtime 의 토대.

---

## 16. 학습 자료

- **The Linux Programming Interface** Ch. 24-28
- **APUE** (Advanced Programming in the UNIX Environment)
- `man 2 fork`, `man 2 execve`, `man 2 wait`, `man 2 clone`

---

## 17. 관련

- [[orphan-zombie]] — wait 누락
- [[signals]] — SIGCHLD
- [[../ipc/pipe-fifo]] — fork + pipe
- [[../virtualization/namespace]] — clone + NEWNS
- [[process]] — Process hub
