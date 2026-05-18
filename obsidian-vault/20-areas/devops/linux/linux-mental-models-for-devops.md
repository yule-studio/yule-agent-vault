---
title: "Linux mental models for DevOps — process / namespace / cgroup / page cache / scheduler"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-18T12:30:00+09:00
tags: [devops, linux, mental-models]
home_hub: devops
related:
  - "[[linux]]"
  - "[[process-management]]"
  - "[[systemd]]"
  - "[[performance-troubleshooting]]"
  - "[[../docker/docker-mental-models]]"
---

# Linux mental models for DevOps — process / namespace / cgroup / page cache / scheduler

**[[linux|↑ linux]]**

---

## 1. 목적

본 문서는 Linux 커널의 5 가지 핵심 모델 (process / namespace / cgroup / page cache / scheduler) 을 DevOps 운영에서 자주 부딪히는 현상의 관점에서 정의한다.

본 문서가 정의하는 것:
- 5 모델 각각의 정의 / 데이터 구조 / 사용자 공간에서의 관찰 방법
- 모델 간 상호작용 (container = namespace + cgroup 등)
- 운영 현상 (OOMKilled / throttling / context switch 폭주 / cache cold start) 의 모델 기반 설명
- 디버깅 시 모델별 1차 신호 위치

본 문서가 정의하지 않는 것:
- 개별 명령 사용법 — [[process-management]], [[networking-basics]], [[performance-troubleshooting]]
- systemd unit 작성 — [[systemd]]
- shell scripting — [[shell-basics]], [[shell-scripting]]

---

## 2. 범위

| 구분 | 포함 |
| --- | --- |
| 대상 커널 | Linux 5.4 이상 (LTS 기준) |
| 대상 init | systemd (대부분의 distro 표준) |
| cgroup | v2 (unified hierarchy). v1 도 일부 언급. |
| 제외 | 커널 내부 자료구조 코드 (`task_struct` 등) — 외부 reference 로 위임 |

---

## 3. 용어

| 용어 | 정의 |
| --- | --- |
| **process** | 커널이 관리하는 실행 단위. `task_struct` 1 개. |
| **thread** | 같은 메모리 / fd 를 공유하는 process group. Linux 에선 `task_struct` 단위로 동등. |
| **PID 1** | init process. systemd / 컨테이너 안 `pid namespace` 의 root. |
| **fd (file descriptor)** | process 가 연 파일 / 소켓 / pipe 의 정수 식별자. |
| **uid / gid** | 권한 단위. user namespace 에서 mapping 됨. |
| **namespace** | 시스템 자원의 view 격리 단위 (mnt / pid / net / uts / ipc / user / cgroup). |
| **cgroup** | resource quota 단위 (cpu / memory / io / pids). |
| **page** | 메모리의 최소 단위 (보통 4KB). |
| **page cache** | 디스크 파일 내용의 메모리 캐시. |

---

## 4. 모델 1 — process

### 4.1 정의

process 는 커널의 `task_struct` 1 개에 대응하는 실행 단위이다. 다음 자원을 갖는다.

| 자원 | 위치 |
| --- | --- |
| 메모리 (가상 주소 공간) | `mm_struct` (heap / stack / mmap) |
| 파일 / 소켓 | `files_struct` (fd table) |
| 권한 | `cred` (uid / gid / capability) |
| signal 핸들러 | `sighand_struct` |
| 작업 디렉터리 / root | `fs_struct` |

### 4.2 생성

process 는 `clone(2)` (또는 `fork(2)`) + `execve(2)` 로 생성된다. `clone` 의 `flags` 가 자식과 부모가 공유할 자원을 결정한다.

| flag | 효과 |
| --- | --- |
| `CLONE_VM` | 메모리 공유 (thread) |
| `CLONE_FILES` | fd 공유 |
| `CLONE_NEWNS` | mount namespace 새로 생성 |
| `CLONE_NEWPID` | pid namespace 새로 생성 |
| `CLONE_NEWUSER` | user namespace 새로 생성 |
| ... | 나머지 namespace flag |

→ 컨테이너 런타임 (runc) 은 `clone` 을 namespace flag 와 함께 호출하여 격리된 process 를 만든다.

### 4.3 상태

| 상태 | 의미 |
| --- | --- |
| R (running) | runqueue 에 있거나 실행 중 |
| S (interruptible sleep) | I/O 대기 (정상) |
| D (uninterruptible sleep) | I/O 대기 (kill 불가) — 길면 디스크 / NFS 문제 |
| Z (zombie) | exit 했으나 부모가 wait 하지 않음 |
| T (stopped) | SIGSTOP / debugger |

### 4.4 관찰

```bash
ps -eo pid,ppid,user,state,comm
top -H                              # thread 단위
cat /proc/<pid>/status              # 메모리 / fd / cap
ls /proc/<pid>/fd/                  # 열린 fd
cat /proc/<pid>/maps                # 가상 주소 매핑
cat /proc/<pid>/limits              # ulimit
strace -p <pid>                     # syscall trace
```

### 4.5 운영 현상

| 현상 | 추적 |
| --- | --- |
| Zombie 누적 | 부모가 `wait()` 안 함 → 부모 재시작 또는 `prctl(PR_SET_CHILD_SUBREAPER)` |
| D state 길게 | iostat / dmesg / NFS mount 확인 |
| fd 누갈 | `ls /proc/<pid>/fd/` 수 증가 → 코드 누갈 / `ulimit -n` 부족 |
| zombie 자식이 container 안에 누적 | container 안 PID 1 이 wait 안 함 — `tini` / `dumb-init` 사용 |

---

## 5. 모델 2 — namespace

### 5.1 정의

namespace 는 시스템 자원의 view 격리 단위이다. 7 종이 존재하며 process 마다 각 종류별로 1 개의 namespace 에 속한다.

| namespace | 격리 자원 | 관찰 |
| --- | --- | --- |
| mnt | mount points | `/proc/<pid>/mounts` |
| pid | process ID space | `/proc/<pid>/ns/pid` |
| net | 인터페이스 / 라우팅 / iptables / 소켓 | `ip netns` |
| uts | hostname / domainname | `uname` |
| ipc | SysV IPC / POSIX MQ | `ipcs` |
| user | UID/GID mapping | `/proc/<pid>/uid_map` |
| cgroup | cgroup root view | `/proc/<pid>/cgroup` |

### 5.2 컨테이너와의 관계

container = 위 7 namespace 를 동시에 새로 생성한 process. 일부만 새로 생성도 가능.

| 옵션 | 결과 |
| --- | --- |
| `docker run --network host` | net namespace 공유 (host 의 port 직접 사용) |
| `docker run --pid host` | pid namespace 공유 (host process 보임) |
| `docker run --ipc host` | ipc namespace 공유 |
| `docker run --userns host` | user namespace 공유 (UID mapping 없음) |

### 5.3 관찰

```bash
lsns                                # 시스템의 모든 namespace
lsns -p <pid>                       # 특정 process 의 namespace
ls -la /proc/<pid>/ns/              # symlink → "ns_type:[inode]"
nsenter -t <pid> -n ip a            # 특정 process 의 net namespace 진입
unshare --pid --fork --mount-proc bash   # 새 pid namespace 의 shell
```

### 5.4 운영 현상

| 현상 | 추적 |
| --- | --- |
| container 안 hostname 변경이 host 영향 X | uts namespace 격리 |
| container 안 PID 1 죽으면 container 종료 | pid namespace 의 root 사망 → 모든 자식 SIGKILL |
| container 안 mount 가 host 에 안 보임 | mnt namespace |
| `docker exec` 가 새 process 띄움 | 기존 namespace 에 join (`setns`) |
| rootless container 가 root 으로 보임 | user namespace 의 UID mapping (0 → 100000 등) |

---

## 6. 모델 3 — cgroup (v2)

### 6.1 정의

cgroup 은 process group 의 resource quota 단위이다. cgroup v2 는 단일 hierarchy (`/sys/fs/cgroup/`) 로 통합되었다.

| controller | 제한 항목 | key 파일 |
| --- | --- | --- |
| cpu | 사용량 / 가중치 | `cpu.max`, `cpu.weight`, `cpu.stat` |
| memory | RSS / cache / swap | `memory.max`, `memory.current`, `memory.events` |
| io | IOPS / bandwidth | `io.max`, `io.stat` |
| pids | PID 수 | `pids.max`, `pids.current` |

### 6.2 cgroup 트리

cgroup 은 트리 구조이며 자식은 부모의 limit 를 초과할 수 없다.

```
/sys/fs/cgroup/
├── system.slice/                       ← systemd 가 관리
│   ├── docker.service/
│   ├── docker-<container_id>.scope/    ← 각 container
│   ├── kubepods.slice/                 ← k8s (kubelet 이 관리)
│   │   ├── burstable.slice/
│   │   │   └── pod<uid>.slice/
│   │   │       └── <container_id>.scope/
│   │   └── besteffort.slice/
│   └── ...
└── user.slice/
    └── user-1000.slice/                ← 사용자별
```

### 6.3 관찰

```bash
systemd-cgls                            # cgroup 트리
systemd-cgtop                           # cgroup 별 사용량
cat /sys/fs/cgroup/<path>/cpu.max       # quota
cat /sys/fs/cgroup/<path>/memory.events # OOM / throttle 카운트
```

### 6.4 운영 현상

#### 6.4.1 OOMKilled

| 단계 | 동작 |
| --- | --- |
| 1 | container 가 `memory.max` 초과 |
| 2 | 커널이 OOM killer 호출 |
| 3 | cgroup 안에서 score 가장 높은 process 선정 (보통 PID 1) |
| 4 | SIGKILL → exit 137 (128 + 9) |
| 5 | k8s / docker 가 status 를 `OOMKilled` 로 기록 |

```bash
dmesg | grep -i 'killed process'
cat /sys/fs/cgroup/<path>/memory.events
```

#### 6.4.2 CPU throttling

| 단계 | 동작 |
| --- | --- |
| 1 | cgroup `cpu.max = "200000 100000"` (100ms 중 200ms — 2 코어) |
| 2 | container 가 quota 초과 |
| 3 | 다음 period 까지 schedule 대기 |
| 4 | 응답 latency 증가, throughput 정체 |

```bash
cat /sys/fs/cgroup/<path>/cpu.stat
# nr_throttled, throttled_usec 가 증가하면 throttling 발생 중
```

→ k8s 에서 `limits.cpu` 만 설정하고 `requests.cpu` 미설정 시 자주 발생.

### 6.5 cgroup v1 과의 차이

| 항목 | v1 | v2 |
| --- | --- | --- |
| hierarchy | controller 별 별도 | 단일 unified |
| memory + swap | 별도 controller | 단일 controller |
| OOM 정책 | global | per-cgroup |
| 사용 | 구 distro | 최신 distro (RHEL 9 / Ubuntu 22.04+) |

---

## 7. 모델 4 — page cache (메모리)

### 7.1 정의

page cache 는 디스크 파일 내용의 메모리 캐시이다. `free` 명령의 `buff/cache` 가 이에 해당.

| 종류 | 설명 |
| --- | --- |
| anon page | 익명 메모리 (heap / stack) — swap 대상 |
| file-backed page | 파일 mmap / read 결과 — 디스크와 동기화 |
| dirty page | 변경되었으나 아직 디스크 미반영 |
| clean page | 디스크와 동일 — 메모리 부족 시 즉시 회수 |

### 7.2 메모리 회수 정책

메모리 부족 시 커널이 다음 순서로 회수한다.

| 우선순위 | 회수 대상 |
| --- | --- |
| 1 | clean page cache (즉시 회수) |
| 2 | dirty page cache (writeback 후 회수) |
| 3 | anon page → swap (활성화 시) |
| 4 | OOM killer 호출 |

### 7.3 관찰

```bash
free -h                                 # available 이 진짜 가용 메모리
cat /proc/meminfo                       # 상세
vmstat 1                                # si/so (swap in/out), bi/bo (block in/out)
sar -B 1                                # page fault 통계
```

### 7.4 운영 현상

| 현상 | 추적 |
| --- | --- |
| `free` 에 메모리 거의 0 인데 정상 동작 | page cache 는 회수 가능 — `available` 확인 |
| cold start 후 응답 느림 | page cache 미적재 → 첫 요청 disk read |
| swap 사용 폭주 | working set > RAM, anon page 가 swap 으로 — latency 폭증 |
| OOMKilled 인데 free 에 buff/cache 충분 | container 의 `memory.max` 가 cgroup 한정 → host page cache 무관 |

→ 컨테이너 OOM 은 host 의 free 메모리와 관련 없다. cgroup limit 만 본다.

### 7.5 dirty page 와 fsync

| 매개변수 | 의미 |
| --- | --- |
| `vm.dirty_ratio` | 시스템 전체 메모리 중 dirty page 가 이 % 초과 시 write blocking |
| `vm.dirty_background_ratio` | 백그라운드 writeback 시작 임계 |
| `vm.dirty_expire_centisecs` | dirty page 가 이 시간 후 writeback 대상 |

→ DB 가 fsync 호출 시 OS dirty page 강제 flush. 디스크 IOPS 가 fsync latency 결정.

---

## 8. 모델 5 — scheduler

### 8.1 정의

Linux 의 스케줄러는 다음 클래스를 가진다.

| 클래스 | 사용 |
| --- | --- |
| SCHED_OTHER (CFS) | 기본 — fair share |
| SCHED_BATCH | CPU bound batch |
| SCHED_IDLE | 최저 우선순위 |
| SCHED_FIFO / RR | real-time (kernel preempt 우선) |
| SCHED_DEADLINE | 마감 시간 보장 |

### 8.2 CFS (Completely Fair Scheduler)

CFS 는 각 task 의 vruntime (가중치 기반 누적 실행 시간) 이 가장 작은 task 를 다음 실행한다. `nice` 값이 가중치를 변경.

| nice | weight | 효과 |
| --- | --- | --- |
| -20 | 88761 | 최대 우선순위 |
| 0 | 1024 | 기본 |
| 19 | 15 | 최저 |

### 8.3 cgroup 의 CPU 통합

cgroup v2 의 `cpu.weight` 가 CFS 의 task 가중치에 매핑된다. `cpu.max` 는 별도의 quota / period bandwidth control 메커니즘.

### 8.4 관찰

```bash
top                                    # %CPU, %st (steal)
mpstat -P ALL 1                        # CPU 별
pidstat 1                              # process 별
perf top                               # 함수 단위 hot spot
cat /proc/<pid>/sched                  # vruntime, switches
```

### 8.5 운영 현상

| 현상 | 추적 |
| --- | --- |
| 응답 latency p99 만 튐 | context switch 폭주 (vmstat 의 cs) |
| %sys 높음 | syscall 폭주 → strace / perf |
| %wa (I/O wait) 높음 | iostat / dirty page flush |
| %st (steal) 높음 | hypervisor 의 다른 VM 이 CPU 점유 (cloud noisy neighbor) |
| CPU 절반만 사용 | 단일 thread 의존 → CPU affinity / GOMAXPROCS |
| container CPU throttling | §6.4.2 |

---

## 9. 모델 간 상호작용

### 9.1 컨테이너 = process + namespace + cgroup

container 의 정의는 다음 3 요소의 동시 적용이다.

```
container =
    clone(CLONE_NEWNS | CLONE_NEWPID | CLONE_NEWNET | ...)
    + cgroup write (cpu.max / memory.max / ...)
    + capability drop (CAP_*)
    + execve(<entrypoint>)
```

→ "container 가 가벼운 이유" = VM 처럼 별도 OS 가 아니라 같은 kernel 의 process 이기 때문. namespace 는 view 격리, cgroup 은 자원 격리, capability 는 권한 격리.

### 9.2 page cache 와 컨테이너

page cache 는 host 의 단일 cache 에서 공유된다. 같은 image 의 동일 파일을 mmap 하는 컨테이너 N 개는 같은 page 를 공유 (copy-on-write).

→ 같은 base image 의 컨테이너를 많이 띄울수록 메모리 footprint 가 효율적.

### 9.3 scheduler 와 cgroup

cgroup `cpu.weight` 는 CFS 의 task 가중치로 변환된다. cgroup tree 안에서 sibling 사이의 가중치 분배가 일어난다.

```
parent cgroup (cpu.weight = 100)
├── child A (cpu.weight = 100)  → 부모 시간의 1/2
└── child B (cpu.weight = 100)  → 부모 시간의 1/2
```

### 9.4 OOM 의 책임 범위

| OOM 발생 위치 | 영향 |
| --- | --- |
| cgroup 안 (container OOM) | 해당 container 의 process kill, 다른 container 무관 |
| host global OOM | 시스템 전체 score 기반 kill, 호스트 안정성 위협 |

→ 모든 production 워크로드는 cgroup limit 필수. limit 없으면 한 워크로드가 host global OOM 유발 가능.

---

## 10. 디버깅 시 모델별 1차 신호 위치

| 증상 | 1차 신호 | 모델 |
| --- | --- | --- |
| process 응답 없음 | `ps`, `kill -0`, `/proc/<pid>/status` 의 State | process |
| 컨테이너 즉시 종료 (exit 137) | `dmesg | grep -i 'killed'`, `memory.events` | cgroup + memory |
| 컨테이너 응답 느려짐 | `cpu.stat` 의 `nr_throttled` | cgroup + scheduler |
| 호스트 응답 전체 느림 | `top` 의 %st, %wa, vmstat 의 si/so | scheduler / page cache |
| 네트워크 격리 동작 안 함 | `lsns -t net -p <pid>` | namespace |
| systemd 서비스 자꾸 죽음 | `journalctl -u <svc>`, `systemctl status <svc>` | process + cgroup |
| 디스크 가득 (but df 정상) | `du`, inode (`df -i`) | filesystem (별도 모델) |

---

## 11. 흔한 실패 모드

| 실패 | 원인 | 모델로부터의 설명 |
| --- | --- | --- |
| 컨테이너 안 PID 1 이 child 미수집 → zombie | container 의 init 미적용 | §4.5 — `tini` / `dumb-init` |
| OOMKilled 인데 host free 충분 | cgroup `memory.max` 한정 | §6.4.1, §7.4 |
| `limits.cpu` 만 있고 latency 폭증 | `cpu.max` quota → throttling | §6.4.2 |
| `docker run --pid host` 후 보안 사고 | pid namespace 미격리 | §5.2 |
| rootless 컨테이너의 file 권한 이상 | user namespace UID mapping | §5.4 |
| cgroup v1 / v2 혼동으로 limit 미반영 | 구 docker / k8s 가 v1 가정 | §6.5 |
| `vm.swappiness` 0 인데 latency 폭증 | anon page reclaim 없이 OOM 직행 | §7.2 |
| 응답 latency 가 부정기적으로 튐 | cloud VM 의 %st (noisy neighbor) | §8.5 |
| 같은 image 100 컨테이너에 OOM | page cache 공유 모르고 limit 산정 | §9.2 |

---

## 12. 참고

- [[linux|↑ linux]]
- [[process-management]]
- [[systemd]]
- [[networking-basics]]
- [[performance-troubleshooting]]
- [[pitfalls]]
- [[../docker/docker-mental-models]]
- [[../kubernetes/kubernetes-mental-models]]
- [[../../computer-science/network/topics/container-networking|↗ CS — container networking (veth / bridge / VXLAN)]] — network namespace 의 실제 활용
- [[../../computer-science/network/tools/iptables-netfilter|↗ CS — iptables / netfilter]] — Linux packet filter
- [[../../computer-science/network/topics/tcp-socket-ops-pitfalls|↗ CS — TCP socket 운영 pitfall]] — TIME_WAIT / ephemeral port / SYN backlog
- Brendan Gregg — Systems Performance (2nd ed) — page cache, scheduler
- man 7 namespaces, man 7 cgroups
- kernel docs — `Documentation/admin-guide/cgroup-v2.rst`
