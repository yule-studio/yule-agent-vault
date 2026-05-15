---
title: "Process management — ps / top / kill / signal"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T06:43:00+09:00
tags: [devops, linux, process]
---

# Process management — ps / top / kill / signal

**[[linux|↑ linux]]**

---

## 1. 프로세스 보기

```bash
ps aux                    # 전체 (BSD style)
ps -ef                    # 전체 (SysV style)
ps -ef --forest           # tree
ps -p 1234 -o pid,ppid,user,cmd
pgrep -f nginx            # nginx 매칭 pid
pidof nginx               # 정확히 nginx pid

# 실시간
top                       # 기본
htop                      # 향상
btop                      # 더 예쁨
glances                   # 종합

# 특정 사용자
ps -u alice
```

---

## 2. ps 출력 컬럼

```
USER       PID  %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
alice    12345  10.0  2.5 200000 50000 pts/0    Ss+  10:00   0:05 /usr/bin/python app.py
```

| 컬럼 | 의미 |
| --- | --- |
| **PID** | process ID |
| **PPID** | parent PID |
| **%CPU** | CPU 사용률 |
| **%MEM** | 메모리 사용률 |
| **VSZ** | virtual memory (kB) |
| **RSS** | resident set (실제 메모리) |
| **STAT** | R(running) S(sleeping) D(uninterruptible) Z(zombie) T(stopped) |
| **TIME** | CPU 누적 시간 |

→ `STAT` 의 `+` = foreground, `s` = session leader.

---

## 3. signal

```bash
kill -l                   # signal 목록

# 자주 쓰는
kill -SIGTERM 1234        # 정상 종료 (cleanup 가능)
kill -SIGKILL 1234        # 강제 (cleanup X)
kill -SIGHUP 1234         # config reload (nginx 등)
kill -SIGSTOP / SIGCONT   # 일시정지 / 재개
kill -9 1234              # SIGKILL 단축
kill 1234                 # 기본 = SIGTERM

# 이름으로
pkill -f "python app.py"
killall nginx
```

| signal | 번호 | 무엇 |
| --- | --- | --- |
| SIGTERM | 15 | graceful (default) — trap 가능 |
| SIGKILL | 9 | force — trap 불가 |
| SIGINT | 2 | Ctrl+C |
| SIGHUP | 1 | terminal 종료 / config reload |
| SIGSTOP | 19 | Ctrl+Z 류 |
| SIGUSR1/2 | | app 정의 |

→ **SIGTERM → wait 30s → SIGKILL** (graceful shutdown).

---

## 4. job control (bash)

```bash
long-command &            # background 실행
jobs                      # 현재 shell 의 job
fg %1                     # foreground 로
bg %1                     # background 재개
Ctrl+Z                    # stop (SIGTSTP)
disown %1                 # shell 종료해도 살아있음
nohup long-cmd &          # HUP 무시 + background
nohup long-cmd > out.log 2>&1 &
```

→ **shell 끊겨도 살리려면 `nohup` 또는 `tmux` / `screen`**.

---

## 5. top / htop 단축키

```
top:
  P    CPU 정렬
  M    Memory 정렬
  k    kill (PID 입력)
  1    CPU per-core
  H    thread 보기
  q    종료

htop:
  F2   설정
  F3   검색
  F4   filter
  F9   kill
  F10  종료
  /    검색
  Space  표시
```

---

## 6. resource 제한

```bash
# 현재 limits
ulimit -a
ulimit -n                 # open file 개수
ulimit -u                 # max user processes
ulimit -v                 # virtual memory

# 변경 (current shell)
ulimit -n 65535

# 영구 (/etc/security/limits.conf)
* soft nofile 65535
* hard nofile 65535

# systemd
LimitNOFILE=65535         # unit file 안에
```

→ "Too many open files" → `ulimit -n` 부족.

---

## 7. nice / ionice (priority)

```bash
# CPU priority (-20 ~ 19, 낮을수록 우선)
nice -n 10 long-cmd       # 낮은 우선순위로 실행
renice -n 15 -p 1234      # 실행 중 변경

# IO priority
ionice -c 3 backup-cmd    # idle class
ionice -c 2 -n 7 long     # best-effort, 낮은 우선
```

---

## 8. zombie / orphan

```
Zombie (Z) — 부모가 wait() 안 한 죽은 process
  → 부모를 kill 또는 부모가 wait() 호출

Orphan — 부모가 먼저 죽음 → init(PID 1) 이 입양
  → 문제 X (init 이 정리)
```

→ container 에서 PID 1 이 wait() 안 하면 zombie 누적 → **tini** / **dumb-init** 사용.

---

## 9. file descriptor (fd)

```bash
ls -l /proc/<pid>/fd/                # 해당 process 의 열린 파일
lsof -p 1234                          # 동일
lsof -i :8080                         # 8080 port 사용 process
lsof /var/log/app.log                 # 해당 파일 사용 process

# socket
ss -tnp | grep :8080
```

→ "address already in use" → `lsof -i :PORT`.

---

## 10. cgroup (container 근간)

```bash
# v2 (현재 표준)
ls /sys/fs/cgroup/                    # cgroup tree
systemd-cgls                          # systemd unit 별
systemd-cgtop                         # top 처럼

# 특정 process 의 cgroup
cat /proc/<pid>/cgroup
```

→ Docker / k8s 가 cgroup 으로 CPU/Mem 제한.

---

## 11. graceful shutdown 패턴

```bash
#!/usr/bin/env bash
PID=$(pidof myapp)
kill -SIGTERM $PID
for i in {1..30}; do
    kill -0 $PID 2>/dev/null || break
    sleep 1
done
kill -SIGKILL $PID 2>/dev/null    # 30초 후 강제
```

→ k8s `terminationGracePeriodSeconds` 와 동일 개념.

---

## 12. 함정

1. **SIGKILL 남발** — cleanup 안 됨 (DB 트랜잭션, file flush).
2. **PID 1 (container)** 이 signal 무시 — `tini` 사용.
3. **ulimit -n 1024** (default) — 많은 connection 시 부족.
4. **zombie 누적** — container PID 1 가 wait X.
5. **nohup 없이 long task** — terminal 끊김 시 죽음.
6. **`kill -9` 으로 nginx reload** — config reload 는 SIGHUP.

---

## 13. 관련

- [[linux|↑ linux]]
- [[systemd]]
- [[shell-basics]]
- [[../docker/concepts|↗ docker]]
