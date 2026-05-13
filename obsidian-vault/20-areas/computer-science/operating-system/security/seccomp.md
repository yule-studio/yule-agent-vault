---
title: "seccomp — Syscall Filter"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T15:50:00+09:00
tags:
  - operating-system
  - security
  - seccomp
---

# seccomp

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | seccomp-bpf / 컨테이너 |

**[[security|↑ Security hub]]**

---

## 1. 한 줄

응용이 호출 가능한 **syscall 화이트리스트** — 나머지는 차단.
exploit 시 attack surface 축소.

---

## 2. 두 모드

### 2.1 Strict
```c
prctl(PR_SET_SECCOMP, SECCOMP_MODE_STRICT);
```

`read / write / _exit / sigreturn` 4 개만 허용. 나머지 → SIGKILL.

→ 거의 안 씀 — 너무 제한적.

### 2.2 Filter (seccomp-bpf)
BPF 프로그램으로 정밀 필터.

---

## 3. seccomp-bpf

```c
#include <linux/seccomp.h>
#include <linux/filter.h>

struct sock_filter filter[] = {
    BPF_STMT(BPF_LD+BPF_W+BPF_ABS, offsetof(struct seccomp_data, nr)),
    BPF_JUMP(BPF_JMP+BPF_JEQ+BPF_K, __NR_read, 0, 1),
    BPF_STMT(BPF_RET+BPF_K, SECCOMP_RET_ALLOW),
    // ... 다른 syscall
    BPF_STMT(BPF_RET+BPF_K, SECCOMP_RET_KILL),
};

struct sock_fprog prog = { .len = ..., .filter = filter };

prctl(PR_SET_NO_NEW_PRIVS, 1);
seccomp(SECCOMP_SET_MODE_FILTER, 0, &prog);
```

직접 BPF — 매우 복잡. 보통 라이브러리 사용 (libseccomp).

---

## 4. libseccomp

```c
#include <seccomp.h>

scmp_filter_ctx ctx = seccomp_init(SCMP_ACT_KILL);
seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(read), 0);
seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(write), 0);
seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit_group), 0);
seccomp_load(ctx);
seccomp_release(ctx);
```

훨씬 단순. 정책 export / import.

---

## 5. Action

| Action | 의미 |
| --- | --- |
| `SCMP_ACT_ALLOW` | 허용 |
| `SCMP_ACT_KILL` / `KILL_PROCESS` / `KILL_THREAD` | 죽임 |
| `SCMP_ACT_ERRNO(EPERM)` | errno 반환 |
| `SCMP_ACT_TRAP` | SIGSYS |
| `SCMP_ACT_TRACE` | ptrace 알림 |
| `SCMP_ACT_LOG` | log 만 |
| `SCMP_ACT_NOTIFY` (5.0+) | userspace notify (seccomp-unotify) |

---

## 6. 컨테이너 / Docker

기본 docker default profile = 약 60 개 syscall 차단:
- `mount`, `umount`, `unshare`
- `reboot`, `kexec_load`
- `keyctl`
- `bpf` (일부)
- `personality`
- ...

```bash
docker run --security-opt seccomp=profile.json ...
docker run --security-opt seccomp=unconfined ...    # ⚠️ 비활성
```

profile.json:
```json
{
    "defaultAction": "SCMP_ACT_ERRNO",
    "syscalls": [
        {
            "names": ["read", "write", ...],
            "action": "SCMP_ACT_ALLOW"
        }
    ]
}
```

---

## 7. K8s

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault           # docker default 비슷
    # 또는
    type: Localhost
    localhostProfile: profiles/myprofile.json
```

전체 cluster 기본:
```
--seccomp-default
```

kubelet 옵션. 1.27+ 기본 활성 검토.

---

## 8. seccomp-unotify (5.0+)

```
응용이 syscall 호출 → seccomp 가 userspace handler 에 알림 → handler 가 결정
```

→ runtime 정책 / 호환 layer (예: gVisor).

---

## 9. NoNewPrivileges 와의 조합

```c
prctl(PR_SET_NO_NEW_PRIVS, 1);
// 이후 exec 시 setuid binary 도 권한 획득 X
```

seccomp 만 + setuid binary = filter 우회 위험. NoNewPrivileges 필수.

systemd:
```ini
NoNewPrivileges=yes
SystemCallFilter=@system-service
```

---

## 10. systemd 의 SystemCallFilter

```ini
[Service]
SystemCallFilter=@system-service
SystemCallFilter=~@privileged @resources
SystemCallErrorNumber=EPERM
NoNewPrivileges=yes
```

systemd 가 미리 정의한 group:
- `@system-service` — 일반 서비스
- `@privileged` — 위험
- `@network-io`
- `@process`
- `@ipc`
- ...

---

## 11. 응용 예 — Chrome / Firefox

Chrome 의 sandbox:
- broker process — root 가깝게
- worker process — seccomp + namespace + chroot + setuid X
- worker 가 file / network 필요 → broker 에 RPC

→ exploit 가 worker 에 있어도 시스템 접근 X.

---

## 12. 프로파일 생성 도구

### 12.1 strace + seccomp-gen

```bash
strace -c -e trace=all ./myapp
# 호출한 syscall 목록 → 허용 list 만들기
```

### 12.2 oci-seccomp-bpf-hook
Docker 등의 자동 profile 학습.

### 12.3 Falco / tracee
eBPF 기반 — syscall 사용 분석.

---

## 13. 단점 / 함정

### 13.1 syscall 표면이 커널마다
새 kernel = 새 syscall. profile 갱신.

### 13.2 응용 동작 차단
미세 호환 — 일부 응용 fail. permissive 모드 / log 로 학습.

### 13.3 wrapper 라이브러리의 추가 syscall
glibc 의 fallback (예: openat 대신 open). 추적.

### 13.4 NoNewPrivileges 누락
setuid binary 가 filter 우회.

### 13.5 ptrace 와 충돌
profile 이 ptrace 차단 → 디버거 안 됨.

### 13.6 io_uring 의 syscall 표면
SQE 안의 op 가 filter 우회 가능 → io_uring 차단 검토.

### 13.7 strict mode 너무 좁음
실용 X.

---

## 14. 학습 자료

- **kernel.org** `Documentation/userspace-api/seccomp_filter.rst`
- **libseccomp** github
- **Chrome Sandbox design**
- **Docker default seccomp profile** (github)
- **systemd.exec(5)** — SystemCallFilter

---

## 15. 관련

- [[capabilities]]
- [[sandbox]]
- [[../syscall/syscall]]
- [[security]] — Security hub
