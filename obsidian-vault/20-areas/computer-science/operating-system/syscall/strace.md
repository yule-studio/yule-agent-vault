---
title: "strace — 시스템 콜 디버거"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T14:00:00+09:00
tags:
  - operating-system
  - syscall
  - strace
  - debug
---

# strace

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 사용 가이드 |

**[[syscall|↑ Syscall hub]]**

---

## 1. 한 줄

응용이 호출하는 모든 syscall + 인자 + 반환값 출력. **OS 레벨 디버그의 표준 도구**.

---

## 2. 기본

```bash
strace ./app
strace ls /tmp

# 기존 프로세스
strace -p $PID
strace -p $PID -p $PID2

# fork 따라가기
strace -f ./app
strace -ff -o trace.log ./app    # PID 별 파일
```

---

## 3. 필터링

```bash
# 특정 syscall 만
strace -e trace=read,write,openat ./app
strace -e trace=network ./app
strace -e trace=file ./app
strace -e trace=process ./app    # fork/exec/wait/exit
strace -e trace=signal ./app
strace -e trace=ipc ./app
strace -e trace=memory ./app
strace -e trace=desc ./app        # FD

# 제외
strace -e trace=!read,write ./app
```

---

## 4. 시간 / 통계

```bash
strace -tt -T ./app        # absolute time + duration
strace -r ./app             # relative time
strace -c ./app             # summary at end (호출 횟수 / 시간)
```

### 4.1 -c 출력 예

```
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 50.10    0.000123          12        10           read
 30.05    0.000074           7        10           write
 10.02    0.000025          25         1           connect
 ...
```

→ 어떤 syscall 이 시간 / 횟수 많은지 빠른 인식.

---

## 5. 상세 출력

```bash
strace -v -s 256 ./app     # 인자 verbose + string 256 자
strace -y ./app             # FD → path
strace -yy ./app             # 더 자세
strace -k ./app             # backtrace (-G 시 컴파일 필요)
strace -i ./app             # 명령 주소
strace -X raw ./app          # raw 모드 (전체)
```

---

## 6. 출력 예

```
openat(AT_FDCWD, "/etc/passwd", O_RDONLY) = 3
read(3, "root:x:0:0:root:/root:/bin/bash\n"..., 4096) = 1234
write(1, "root:x:0:0:root:/root:/bin/bash\n"..., 1234) = 1234
close(3) = 0
```

각 줄:
- syscall 이름
- 인자
- 반환값 (또는 `-1 ENOENT (No such file or directory)`)

---

## 7. 진단 시나리오

### 7.1 "왜 안 됨" — 설정 파일 / FD 누락

```bash
strace -e trace=openat,read ./app 2>&1 | grep ENOENT
```

→ 응용이 어떤 설정 파일 찾는지 / 못 찾는지.

### 7.2 hang / 느림

```bash
strace -p $PID
# 멈춰 있는 syscall 보임 (예: read 대기)
```

### 7.3 시그널

```bash
strace -e trace=signal -p $PID
# SIGTERM / SIGSEGV 시점 확인
```

### 7.4 네트워크

```bash
strace -e trace=network -p $PID
# socket / connect / send / recv
```

### 7.5 fork 디버그

```bash
strace -f -e trace=process ./app
```

자식 process 의 syscall 까지 따라감.

---

## 8. ltrace — 라이브러리 호출

```bash
ltrace ./app           # malloc, free, strcpy 등
ltrace -c ./app
ltrace -p $PID
```

strace 가 syscall, ltrace 가 user library function.

---

## 9. perf trace

```bash
perf trace ./app
perf trace -p $PID
```

strace 대비 overhead 적음. summary 도 빠름.

---

## 10. eBPF 기반

```bash
# bpftrace
bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s\n", str(args->filename)); }'

# bcc — execsnoop / opensnoop
opensnoop
execsnoop
```

자세히 → 별도 (ebpf 노트는 linux 폴더로)

---

## 11. dtrace / dtruss (macOS / BSD)

```bash
sudo dtruss ./app
sudo dtruss -p $PID
```

macOS 의 strace 동등. SIP 비활성 필요할 수도.

---

## 12. strace 의 overhead

```
syscall 마다 trap (ptrace) → 응용 매우 느림 (수 배~수십 배)
```

→ profiling 보다 디버깅 도구. perf / eBPF 가 운영 친화.

---

## 13. 보안 / 권한

- root 또는 같은 UID + 추가 cap (`CAP_SYS_PTRACE`)
- `kernel.yama.ptrace_scope` (Ubuntu): 1 부터 부모만, 2 root 만

```bash
sysctl kernel.yama.ptrace_scope
```

container 안에서 다른 process strace = 권한 필요.

---

## 14. 자주 보는 함수 / 의미

```
openat(AT_FDCWD, "...", O_RDONLY)            파일 열기
read/write(fd, "...", N) = M                  M byte 처리
mmap(NULL, sz, PROT_READ, MAP_PRIVATE, ...)   매핑
brk(NULL) = ...                                heap 시작
brk(0x...) = ...                               heap 확장 (malloc)
clone(...)                                     fork / pthread_create
execve(...)                                    프로그램 시작
futex(uaddr, FUTEX_WAIT, val, NULL)            mutex 대기
poll(...) = N                                  I/O multiplexing
nanosleep({0,N}, NULL) = 0                     sleep
```

---

## 15. 함정

### 15.1 overhead
운영 사용 X — debug 만.

### 15.2 strace 가 응용 동작 변화
TIMing 다름 → race condition 사라짐 (Heisenbug).

### 15.3 출력 너무 많음
filter (`-e`) + `-s 256`.

### 15.4 stderr 가 trace
응용 stderr 와 섞임. `-o file` 사용.

### 15.5 multi-process
`-f` 필수.

### 15.6 hang process strace 후 detach
strace 종료 시 응용 재개. 가끔 자식 process 죽음.

### 15.7 sandbox / seccomp 환경
strace 가 자체로 syscall 추가 → seccomp violation 가능.

---

## 16. 학습 자료

- **strace.io** — 공식
- **The Linux Programming Interface** Ch. 28 (`ptrace`)
- **Brendan Gregg — eBPF / strace 비교**

---

## 17. 관련

- [[posix-calls]]
- [[syscall]] — Syscall hub
