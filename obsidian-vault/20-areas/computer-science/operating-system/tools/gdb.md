---
title: "gdb — GNU Debugger"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:05:00+09:00
tags:
  - operating-system
  - tools
  - gdb
  - debug
---

# gdb

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | gdb 기본 + 자주 쓰는 명령 |

**[[tools|↑ Tools hub]]**

---

## 1. 시작

```bash
gdb ./myapp                              # 새 program
gdb -p $PID                               # 기존 process 에 attach
gdb ./myapp core                          # core dump 분석
gdb --args ./myapp arg1 arg2
gdb -tui ./myapp                          # text UI
```

debug symbol 필요 — `-g` 로 컴파일:
```bash
gcc -g -O0 main.c -o myapp
```

`-O0` 가 디버그 친화 (variable 보임).

---

## 2. 기본 명령

```
run / r            program 시작
continue / c       resume
step / s           한 줄 (함수 진입)
next / n           한 줄 (함수 안 안 들어감)
finish             현재 함수 끝까지
until 30           30 줄까지

break / b main     breakpoint
b file.c:42
b ClassA::method
info break
delete 2
disable 3
enable 3

watch var          값 변경 시 멈춤
rwatch var         읽힘 시
awatch var         R/W 시

backtrace / bt     호출 스택
bt full            + variable
frame 3            stack frame 3 으로
up / down

print var          값
print *p
print arr[0]@10    배열 10 element
print/x var         hex
print/d / o / s / c / f

display var         매 stop 마다 자동 print
info display

list / l           소스 보기
list main
list 1,20

info locals
info args
info registers
info threads
info sharedlibrary

quit / q
```

---

## 3. core dump 분석

```bash
# 활성
ulimit -c unlimited
sysctl kernel.core_pattern=/var/crash/core.%e.%p

# 생성
./crash                                   # SIGSEGV → core

# 분석
gdb ./crash /var/crash/core.crash.1234
(gdb) bt
(gdb) bt full
(gdb) thread apply all bt
(gdb) info registers
```

---

## 4. running process attach

```bash
# 권한 (Ubuntu)
sudo sysctl kernel.yama.ptrace_scope=0
# 또는 sudo gdb
sudo gdb -p $PID
```

```
(gdb) bt
(gdb) thread apply all bt
(gdb) info threads
(gdb) detach
(gdb) continue
```

자세히 → [[../security/sandbox]] (ptrace 권한)

---

## 5. 자주 쓰는 디버그 시나리오

### 5.1 SIGSEGV 위치

```
gdb ./app
(gdb) run
... Program received signal SIGSEGV ...
(gdb) bt
(gdb) print *p          # NULL 인지 확인
```

### 5.2 hang process

```bash
gdb -p $PID
(gdb) thread apply all bt
# futex / read / poll 어디서 멈춤
```

### 5.3 deadlock

```
(gdb) thread apply all bt
# 모든 thread 의 mutex 대기 보기
# 같은 mutex 잡고 다른 거 기다리면 deadlock
```

### 5.4 메모리 손상

```
(gdb) watch *0x12345
(gdb) rwatch ptr
(gdb) c                  # 값 변경 시 멈춤
```

---

## 6. 멀티스레드

```
(gdb) info threads
* 1  Thread 1234 (LWP 1234) ...
  2  Thread 1235 ...

(gdb) thread 2           # thread 2 로 전환
(gdb) bt
(gdb) thread apply all bt
(gdb) thread apply all info locals

# non-stop mode
(gdb) set non-stop on
```

---

## 7. 자주 쓰는 set

```
set print pretty on
set print elements 0       # array / string 전체
set pagination off
set logging file gdb.log
set logging on

set scheduler-locking step  # step 시 다른 thread 정지
set follow-fork-mode child
set detach-on-fork off
```

---

## 8. python scripting

```python
(gdb) python
def all_threads_bt():
    for thread in gdb.selected_inferior().threads():
        thread.switch()
        gdb.execute("bt")
end

(gdb) python all_threads_bt()
```

복잡한 디버그 자동화.

---

## 9. TUI (Text UI)

```
gdb -tui ./app

Ctrl-x a    TUI on/off
Ctrl-x 2    layout regs / asm
Ctrl-x o    focus 변경
Page Up/Dn  소스 스크롤
```

또는 외부:
- `gdb-dashboard` (Python script)
- `cgdb`
- `gef` / `pwndbg` — 보안 / pwn 친화 plugin

---

## 10. rr — Record & Replay

```bash
sudo apt install rr
rr record ./app
rr replay
(rr) reverse-continue
(rr) reverse-step
```

→ 시간 거꾸로 디버깅. heisenbug 잡기.

---

## 11. 다른 언어

### 11.1 Rust

```bash
rust-gdb ./target/debug/myapp
```

또는 lldb.

### 11.2 Go
`dlv` (Delve) 가 표준.

```bash
dlv debug
dlv attach $PID
```

### 11.3 Python
`pdb`, `pudb`, IDE debugger.

### 11.4 Java
jdb, IDE.

### 11.5 Node
`node inspect`, Chrome DevTools.

---

## 12. ASAN / TSAN / MSAN

런타임 sanitizer — bug 자동 발견:

```bash
gcc -fsanitize=address -g main.c -o app
./app
# Address Sanitizer 가 발견한 메모리 오류 report

# 종류
-fsanitize=address       # ASan
-fsanitize=thread        # TSan
-fsanitize=memory        # MSan (uninit)
-fsanitize=undefined     # UBSan
-fsanitize=leak          # LSan
```

dev / CI 에서 매우 강력.

---

## 13. valgrind

```bash
valgrind --leak-check=full ./app
valgrind --tool=memcheck ./app
valgrind --tool=helgrind ./app           # race detector
valgrind --tool=callgrind ./app
```

매우 느림 (10-50x). dev 만.

---

## 14. 함정

### 14.1 release build
optimization 후 변수 / line 정보 부정확. `-O0 -g` 권장.

### 14.2 sym 없는 strip binary
no debug info — `objcopy --add-gnu-debuglink` 또는 `*-dbgsym` 패키지.

### 14.3 PIE / ASLR
주소 매번 다름. gdb 가 자동 처리.

### 14.4 ptrace 권한
`kernel.yama.ptrace_scope`.

### 14.5 fork 디버그
`set follow-fork-mode child` / `set detach-on-fork off`.

### 14.6 signal handling
gdb 가 시그널 가로채. `handle SIGPIPE nostop noprint pass`.

### 14.7 core 가 없음
`ulimit -c unlimited` + `/proc/sys/kernel/core_pattern`.

### 14.8 container 안 gdb
`--cap-add=SYS_PTRACE --security-opt=seccomp=unconfined`.

---

## 15. 학습 자료

- **GDB Documentation** — sourceware.org/gdb
- **rr** — rr-project.org
- **Beej's Guide to C** + debugging
- **The Art of Debugging with GDB** (No Starch)

---

## 16. 관련

- [[lsof]]
- [[perf]]
- [[../syscall/strace]]
- [[tools]] — Tools hub
