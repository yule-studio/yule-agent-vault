---
title: "Linux Namespace — 8 종 격리"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T15:05:00+09:00
tags:
  - operating-system
  - virtualization
  - namespace
  - container
---

# Linux Namespace

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 8 종 namespace |

**[[virtualization|↑ Virt hub]]**

---

## 1. 한 줄

Linux 의 **리소스 격리 메커니즘**. 각 종류의 자원에 대해 "자기만 보는" 뷰를 제공. 컨테이너의 토대.

---

## 2. 8 종

| Namespace | 격리 | 도입 |
| --- | --- | --- |
| **`mnt`** | mount point | 2.4.19 |
| **`pid`** | PID | 2.6.24 |
| **`net`** | network (interface, route, port, iptables) | 2.6.29 |
| **`ipc`** | SysV IPC, POSIX MQ | 2.6.19 |
| **`uts`** | hostname, domainname | 2.6.19 |
| **`user`** | UID, GID, capability | 3.8 |
| **`cgroup`** | cgroup root view | 4.6 |
| **`time`** | system clock (CLOCK_MONOTONIC etc.) | 5.6 |

---

## 3. 만들기 — clone / unshare / setns

### 3.1 clone

```c
int flags = CLONE_NEWNS | CLONE_NEWPID | CLONE_NEWNET | ...;
clone(child_fn, stack, flags, arg);
```

### 3.2 unshare

```bash
sudo unshare -p -f -m -n --mount-proc bash
# 새 PID + mount + network namespace + shell
ps aux                    # 자기 자신만
```

```c
unshare(CLONE_NEWPID | CLONE_NEWNS);
// fork 후 자식이 새 ns 안
```

### 3.3 setns

```c
int fd = open("/proc/1234/ns/net", O_RDONLY);
setns(fd, CLONE_NEWNET);
// 이제 pid 1234 와 같은 network namespace
```

`nsenter` 명령 의 기반.

---

## 4. /proc 으로 보기

```bash
ls -l /proc/$PID/ns/
# cgroup -> 'cgroup:[40259...]'
# ipc    -> 'ipc:[40259...]'
# mnt    -> 'mnt:[40259...]'
# net    -> 'net:[40259...]'
# pid    -> 'pid:[40259...]'
# user   -> 'user:[40258...]'
# uts    -> 'uts:[40259...]'
```

같은 inode 번호 = 같은 namespace.

```bash
lsns                                  # 모든 namespace 목록
lsns -p $PID
nsenter -t $PID -a                     # 모든 ns 에 진입
nsenter -t $PID -n ip addr             # 그 net ns 에서 명령
```

---

## 5. mnt namespace

```
각 ns 가 자기 mount table
한 ns 의 mount 가 다른 ns 에 안 보임
```

- pivot_root / chroot 로 rootfs 변경
- shared / slave / private propagation

```bash
unshare -m bash
mount -t tmpfs none /mnt
ls /mnt        # 자기만 보임
exit
```

---

## 6. pid namespace

```
새 pid ns 의 첫 process = PID 1
중첩 가능 — 부모 ns 에선 다른 PID 로 보임
```

```bash
sudo unshare -p -f --mount-proc bash
echo $$         # 1
ps -ef          # 자기만
```

⚠️ **PID 1 의 책임** — 자식 reap + SIGTERM 처리. 컨테이너의 init 함정.

자세히 → [[../process/orphan-zombie#10-컨테이너의-pid-1-함정]]

---

## 7. net namespace

각 ns = 자기만의 network stack:
- interface (lo, eth0, ...)
- routing table
- iptables / nftables
- socket / port

```bash
ip netns add red
ip netns add blue
ip netns exec red ip link show

# veth pair 로 연결
ip link add veth-red type veth peer name veth-blue
ip link set veth-red netns red
ip link set veth-blue netns blue
ip netns exec red ip addr add 10.0.0.1/24 dev veth-red
ip netns exec red ip link set veth-red up
# ... blue 도 비슷
```

→ 컨테이너 / K8s 네트워크의 토대.

---

## 8. ipc namespace

- SysV shm / sem / msg queue
- POSIX MQ

격리 — 다른 ns 의 IPC 안 보임.

---

## 9. uts namespace

- hostname
- domainname

```bash
unshare -u bash
hostname container1
hostname                # container1
exit
hostname                # 원래
```

---

## 10. user namespace

```
ns 안의 UID → 호스트의 다른 UID 로 매핑
ns 안에서 root (UID 0) = 호스트에선 normal user
```

```c
// /proc/$PID/uid_map
"0 1000 1"   // ns 의 0 ~ 1 → host 의 1000 ~ 1001
```

```bash
unshare -U -r bash
id              # uid=0(root) — 진짜 root 아님!
```

### 10.1 의미
- **rootless container** — 호스트에서 root 권한 없이 container 안 root
- 보안 격리 ↑
- capability 도 user ns 안에서 의미

### 10.2 함정
- 일부 syscall 권한 변경 (mount, network)
- UID/GID 매핑 신중

---

## 11. cgroup namespace

cgroup hierarchy 의 root 가 ns 안에서 다르게 보임 → container 안에서 자기 cgroup 만 보임.

자세히 → [[cgroups]]

---

## 12. time namespace (5.6+)

`CLOCK_MONOTONIC` / `CLOCK_BOOTTIME` 의 offset.

```c
unshare(CLONE_NEWTIME);
// /proc/self/timens_offsets 에 monotonic 12345 0 같은 offset
```

CRIU (checkpoint/restore) — 시간 일관성 유지.

---

## 13. 컨테이너 = namespace + cgroup + ...

```c
clone(init_fn, stack,
      CLONE_NEWNS | CLONE_NEWPID | CLONE_NEWNET |
      CLONE_NEWIPC | CLONE_NEWUTS | CLONE_NEWUSER |
      CLONE_NEWCGROUP,
      arg);
// + cgroup 의 process 등록
// + pivot_root 으로 rootfs 변경
// + capability drop
// + seccomp filter
// + apparmor profile
// → 컨테이너 init 완성
```

이를 추상화한 게 **runc** (OCI). 자세히 → [[container]]

---

## 14. nsenter — 진입

```bash
nsenter -t $PID -a                     # 모든 ns
nsenter -t $PID -n -p -m bash          # 일부
nsenter --target $PID --net --pid bash
```

debug / sidecar / k8s exec 의 기반.

---

## 15. 디버그 예

```bash
# 컨테이너 안 namespace
docker inspect <c> | grep -i pid
sudo lsns -p $PID
sudo nsenter -t $PID -n ip addr
sudo nsenter -t $PID -n ss -tln
```

→ container 안에서 못 보는 것도 host 에서 진입해 확인.

---

## 16. 보안 — namespace 만 X

격리만으로는 부족:
- kernel exploit
- shared kernel data
- side channel

추가 필수:
- seccomp (syscall filter)
- capability drop
- AppArmor / SELinux
- read-only rootfs
- no-new-privileges

자세히 → [[../security/security]]

---

## 17. 함정

### 17.1 PID 1 책임
SIGCHLD reap + SIGTERM. tini / dumb-init.

### 17.2 PID namespace + kill
호스트의 다른 PID 로 kill — 다른 process 죽일 수도. ns 안에서.

### 17.3 user ns + mount
일부 fs (procfs, sysfs) 는 사용자 ns 안 mount 제한.

### 17.4 namespace 의 한계
같은 kernel — 0-day 위험. VM 만큼 격리 X.

### 17.5 net namespace 의 loopback
default 빈 상태 — `ip link set lo up` 필요.

### 17.6 mount propagation
shared 면 한쪽 mount 가 다른 ns 에 전파. systemd 의 함정.

### 17.7 hostname 변경
같은 uts ns 면 모든 process 에 영향.

---

## 18. 학습 자료

- **The Linux Programming Interface** Ch. 51 (옛, namespace 추가)
- `man 7 namespaces`, `man 7 pid_namespaces`, etc.
- **Containers from Scratch** — Liz Rice (블로그 / 강연)
- **runc** 소스

---

## 19. 관련

- [[cgroups]]
- [[container]]
- [[../process/pcb]] — nsproxy
- [[virtualization]] — Virt hub
