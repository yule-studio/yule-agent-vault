---
title: "lsof — List Open Files"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:10:00+09:00
tags:
  - operating-system
  - tools
  - lsof
---

# lsof

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | lsof 사용 가이드 |

**[[tools|↑ Tools hub]]**

---

## 1. 한 줄

"**열린 파일** 모두 보여줘" — process / FD / socket / mount 의 추적.

Linux 의 "모든 것은 파일" 철학 → lsof 가 운영의 만능 도구.

---

## 2. 기본

```bash
sudo lsof | head                          # 모든 (매우 많음)
sudo lsof -u alice                         # 사용자
sudo lsof -p $PID
sudo lsof /var/log/syslog                  # 어떤 process 가
sudo lsof -i                               # 모든 network
sudo lsof -i :8080                          # 포트
sudo lsof -i tcp:443
sudo lsof -i 4 -P                          # IPv4, no service name
sudo lsof -nP                               # no name resolve
```

---

## 3. 출력

```
COMMAND   PID  USER   FD   TYPE     DEVICE  SIZE/OFF    NODE NAME
nginx    1234  www    3u   IPv4    123456     0t0      TCP *:80 (LISTEN)
nginx    1234  www    4u   IPv6    123457     0t0      TCP *:80 (LISTEN)
mysqld   2345  mysql  cwd  DIR     8,2        4096     2 /var/lib/mysql
firefox  3456  alice  mem  REG     8,2     20000     12345 /usr/lib/libc.so.6
```

| FD | 의미 |
| --- | --- |
| `cwd` | current working directory |
| `rtd` | root directory |
| `txt` | program text (binary) |
| `mem` | memory-mapped |
| `mmap` | memory-mapped |
| `0u 1u 2u` | stdin/stdout/stderr |
| `3u` | FD 3 (u=read/write, r=read, w=write) |
| `DEL` | deleted but open |

---

## 4. 자주 보는 시나리오

### 4.1 어떤 process 가 포트 사용?

```bash
sudo lsof -i :8080
# 또는
sudo ss -tlnp | grep 8080
```

### 4.2 누가 파일 잡고 있나

```bash
sudo lsof /path/to/file
sudo fuser -uv /path/to/file
```

→ "Resource busy" / umount 안 됨.

### 4.3 삭제된 파일이 디스크 차지

```bash
sudo lsof | grep deleted
sudo lsof +L1                               # link count < 1 (deleted)
```

→ 어떤 process 가 deleted file 보유 → `df` 디스크 안 줄어듦. process restart.

### 4.4 모든 network 연결

```bash
sudo lsof -i
sudo lsof -i -nP                            # 빠름
sudo lsof -i 4 -P                           # IPv4 only
sudo lsof -i 6 -P
sudo lsof -i tcp -nP
sudo lsof -i udp -nP

# state 별
sudo lsof -i -nP | grep ESTABLISHED
sudo lsof -i -nP | grep LISTEN
```

### 4.5 특정 process 의 모든 FD

```bash
sudo lsof -p $PID
sudo lsof -p $PID -nP

# 또는
ls -l /proc/$PID/fd/
```

자세히 → [[../filesystem/inode-dentry#9-file-descriptor-fd]]

### 4.6 사용자별

```bash
sudo lsof -u alice
sudo lsof -u ^root                          # exclude root
```

### 4.7 디렉토리 안의 모든

```bash
sudo lsof +D /var/log                       # recursive
sudo lsof +d /var/log                        # only that dir
```

---

## 5. 옵션 cheatsheet

```
-p PID          process
-u USER         user
-i [proto:port] network
-n               no DNS reverse
-P               no service name (numeric port)
-c CMD           command (prefix)
-d FD            FD 번호
+D dir           recursive
+L num           link count limit
-t               PID only (xargs friendly)
-r N             repeat every N seconds
```

---

## 6. 응용 예

### 6.1 80/443 listen 중 누구?

```bash
sudo lsof -i :80 -i :443 -nP
```

### 6.2 PostgreSQL 의 모든 connection

```bash
sudo lsof -i :5432 -nP -sTCP:ESTABLISHED
```

### 6.3 큰 log file 누가 보유?

```bash
sudo lsof +L1 -a -c java
```

### 6.4 port 별 process group

```bash
sudo lsof -i -nP | awk '{print $9}' | sort | uniq -c | sort -rn | head
```

### 6.5 file 잡은 모든 process kill

```bash
sudo fuser -k /path/to/file
sudo lsof -t /path/to/file | xargs sudo kill
```

---

## 7. fuser — lsof 의 가벼운 친구

```bash
sudo fuser /path/to/file                    # 어떤 process 가
sudo fuser -v /path/to/file
sudo fuser -k /path/to/file                  # kill
sudo fuser -m /mount/point                   # 마운트 잡은 모든
sudo fuser 8080/tcp                          # 포트
```

mount 해제 시:
```bash
sudo umount /mnt
# umount: /mnt: target is busy
sudo fuser -m /mnt                           # 누가 잡았나
sudo fuser -km /mnt                           # 강제
```

---

## 8. 비교 — netstat / ss

```bash
# 옛 — netstat
netstat -tunap

# 현대 — ss
ss -tunap
ss -tlnp
ss -xp                                       # Unix socket
ss -s                                         # summary
```

자세히 → [[../linux/networking#3-ss-sockets-statistics]]

---

## 9. /proc 으로 직접

```bash
ls -l /proc/$PID/fd/                          # FD 목록
ls -l /proc/$PID/exe                          # binary
ls -l /proc/$PID/cwd                           # cwd
cat /proc/$PID/cmdline | tr '\0' ' '
cat /proc/net/tcp                              # 모든 TCP socket
```

자세히 → [[../linux/proc-sys]]

---

## 10. 권한

- 자기 process = lsof 가능
- 다른 user / 시스템 = sudo
- container 안에서 host 보기 = host namespace 진입 필요

---

## 11. 함정

### 11.1 매우 많은 출력
`-nP` (no DNS / 서비스 name) → 빠름. 필터 (`grep`) + 한정 (`-u`, `-c`, `-i`).

### 11.2 sudo 누락
다른 process 의 FD 못 봄.

### 11.3 lsof 의 overhead
큰 시스템에서 lsof = 수 초 + I/O. 자주 호출 X.

### 11.4 SELinux / AppArmor
일부 fd 보임 제한.

### 11.5 deleted file 의 의미
디스크 회수 = close 까지. process restart.

### 11.6 net namespace
container 의 socket = container 안 또는 nsenter.

### 11.7 mount namespace
container 의 file = `/proc/$PID/root/`.

---

## 12. 학습 자료

- `man lsof` — 매우 자세
- **lsof tutorial** — Daniel Stenberg, ...
- **TCPing / mTR / strace** 와 함께

---

## 13. 관련

- [[../linux/proc-sys]]
- [[../linux/networking]]
- [[gdb]]
- [[tools]] — Tools hub
