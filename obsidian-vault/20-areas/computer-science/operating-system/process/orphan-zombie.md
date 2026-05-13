---
title: "Zombie / Orphan — 자식 프로세스의 잔재"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T09:25:00+09:00
tags:
  - operating-system
  - process
  - zombie
---

# Zombie / Orphan

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Zombie / Orphan / SIGCHLD |

**[[process|↑ Process hub]]**

---

## 1. 두 가지 상황

| 상황 | 부모 | 자식 | 결과 |
| --- | --- | --- | --- |
| **Zombie** | 살아있음, `wait()` 안 함 | exit 했지만 PCB 남음 | 자원 누수 |
| **Orphan** | 죽음 | 살아있음 | init (PID 1) 이 양도받음 |

---

## 2. Zombie — 좀비 프로세스

```c
pid_t pid = fork();
if (pid == 0) {
    // child
    _exit(0);                // 즉시 종료
}
// 부모는 wait() 안 함
sleep(60);                    // 그 동안 자식은 좀비
```

```bash
$ ps aux | grep defunct
me   12345  0.0  ... [child] <defunct>
```

### 2.1 왜 남는가
자식의 exit code / 사용 자원을 부모가 회수할 때까지 PCB 보관.

### 2.2 위험
- PID 공간 (기본 32768) 고갈 → 새 fork 불가
- task_struct 누적

---

## 3. Orphan — 고아 프로세스

```c
pid_t pid = fork();
if (pid == 0) {
    // child
    sleep(60);                // 부모보다 오래 산다
}
_exit(0);                     // 부모 즉시 종료
```

→ 자식의 부모 = **init (PID 1)** 로 재배정. (`getppid()` 가 1).

### 3.1 init 의 책임
init 은 자식 exit 시 자동으로 `wait()` → 좀비 회수.

### 3.2 systemd
PID 1 (systemd) 가 자동 reap. 단, container 안에서는 init 이 없는 경우 주의 (10 절).

---

## 4. Daemon — 의도적 Orphan

```c
pid_t pid = fork();
if (pid > 0) _exit(0);            // 부모 종료 → 자식 = orphan
// 자식 = 새 session / process group
setsid();
chdir("/");
close(0); close(1); close(2);
// 데몬으로 동작
```

또는:

```c
daemon(0, 0);                      // glibc 의 한 줄 helper
```

→ 셸에서 분리. 표준 데몬 패턴.

systemd 시대엔 **Type=simple** (fg 그대로) 가 권장 — 데몬 fork 패턴은 옛.

---

## 5. wait / waitpid 로 회수

```c
pid_t pid = fork();
if (pid > 0) {
    int status;
    waitpid(pid, &status, 0);
    if (WIFEXITED(status))
        printf("exit code %d\n", WEXITSTATUS(status));
}
```

자세히 → [[fork-exec#7-wait--waitpid]]

---

## 6. SIGCHLD 로 자동 회수

```c
void on_chld(int sig) {
    while (waitpid(-1, NULL, WNOHANG) > 0)
        ;                          // 모든 종료 자식 회수
}

struct sigaction sa = {
    .sa_handler = on_chld,
    .sa_flags = SA_RESTART | SA_NOCLDSTOP
};
sigaction(SIGCHLD, &sa, NULL);
```

⚠️ 루프 + `WNOHANG` 필수 — 한 시그널 동안 여러 자식 죽을 수 있음.

---

## 7. SIG_IGN — 자동 reap

```c
signal(SIGCHLD, SIG_IGN);
// 또는
struct sigaction sa = { .sa_handler = SIG_DFL, .sa_flags = SA_NOCLDWAIT };
sigaction(SIGCHLD, &sa, NULL);
```

→ POSIX: SIGCHLD 를 IGN 으로 설정하거나 SA_NOCLDWAIT 면 자동 reap.
exit code 못 받음 — 신경 안 쓰는 fire-and-forget 만.

---

## 8. 진단

### 8.1 좀비 찾기

```bash
ps aux | awk '$8 ~ /Z/'
ps -eo pid,ppid,state,comm | grep ' Z '
```

### 8.2 부모 확인

```bash
ps -o ppid= -p <zombie_pid>
```

→ 그 부모가 wait 누락한 코드. 부모 SIGCHLD / wait 추가 또는 부모 종료.

### 8.3 부모 죽으면

좀비가 init 으로 reparent → 즉시 reap.

```bash
kill -9 <parent_pid>
# 좀비 자동 사라짐
```

---

## 9. PID 고갈

```bash
cat /proc/sys/kernel/pid_max         # 기본 32768 또는 4194304
echo 4194304 > /proc/sys/kernel/pid_max
```

좀비 누적 → PID 고갈 → 새 fork 실패 (`ENOMEM` / `EAGAIN`). 사고의 신호.

---

## 10. 컨테이너의 PID 1 함정 ⚠️

Docker 컨테이너 안에서 메인 프로세스가 **PID 1** 이 됨.
PID 1 의 책임:
1. SIGCHLD 받고 reap
2. SIGTERM 핸들링 (받지 않으면 graceful X)

문제:
- 일반 앱은 PID 1 책임 모름 → 좀비 누적 / shutdown 무시.

해결:
- `docker run --init` — `tini` 가 PID 1 으로 시작
- 또는 dumb-init / tini 직접 사용
- Kubernetes 의 `shareProcessNamespace` 도 검토

---

## 11. 시나리오

### 11.1 좀비 누적 — 디버그

```bash
$ ps aux | grep defunct | wc -l
1024

$ ps -o pid,ppid,state,comm | awk '$3 ~ /Z/ {print $2}' | sort -u
1234
```

→ PID 1234 부모. 그 부모의 코드를 보고 SIGCHLD / wait 누락 수정.

### 11.2 데몬화

```c
if (fork() != 0) _exit(0);     // 부모 종료
setsid();
if (fork() != 0) _exit(0);     // session leader 분리
umask(0);
chdir("/");
for (int fd=getdtablesize()-1; fd>=0; fd--) close(fd);
open("/dev/null", O_RDWR);     // stdin
dup(0); dup(0);                 // stdout, stderr
```

### 11.3 systemd 시대의 권장

```ini
[Service]
Type=simple        # fork 하지 않음 — 직접 실행
ExecStart=/usr/bin/myapp
Restart=on-failure
```

→ 더 단순 + 안전.

---

## 12. 함정

### 함정 1 — wait 없는 fork
좀비 누적.

### 함정 2 — SIGCHLD 핸들러에서 wait 1번만
N 동시 종료 시 좀비 남음. 루프 + WNOHANG.

### 함정 3 — `signal(SIGCHLD, SIG_IGN)` 의 OS 의존
POSIX 표준이지만 일부 옛 UNIX 에선 무시 안 함. sigaction + SA_NOCLDWAIT 권장.

### 함정 4 — 컨테이너 PID 1
앱이 자식 reap / SIGTERM 무시. tini 사용.

### 함정 5 — Daemon 의 옛 패턴
systemd Type=simple 이 더 단순.

### 함정 6 — 멀티스레드 + fork + zombie
fork 호출 스레드만 살아남음. 다른 스레드의 wait 호출자 사라짐.

### 함정 7 — `kill -9` 한 좀비
이미 죽음. 부모를 죽여야 사라짐.

---

## 13. 학습 자료

- **The Linux Programming Interface** Ch. 26
- `man 2 wait`, `man 7 signal`
- **tini** — github.com/krallin/tini
- **dumb-init** — github.com/Yelp/dumb-init

---

## 14. 관련

- [[fork-exec]] — 생성
- [[signals]] — SIGCHLD
- [[states]] — Z 상태
- [[process]] — Process hub
