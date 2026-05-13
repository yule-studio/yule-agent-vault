---
title: "cgroups — Control Groups (v1 / v2)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T15:10:00+09:00
tags:
  - operating-system
  - virtualization
  - cgroup
---

# cgroups

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | v1 / v2 / controller |

**[[virtualization|↑ Virt hub]]**

---

## 1. 한 줄

프로세스 그룹에 대해 **CPU / 메모리 / I/O / network** 자원 제한 + 통계 + 격리.

Google (2007) → 2.6.24 mainline. 컨테이너의 자원 관리 토대.

---

## 2. v1 vs v2

| | v1 | v2 |
| --- | --- | --- |
| 도입 | 2007 | 2016 (mainline) |
| 구조 | controller 별 독립 hierarchy | 단일 통합 hierarchy |
| mount | `/sys/fs/cgroup/<ctrl>/` | `/sys/fs/cgroup/` |
| 일관성 | 복잡 | 단순 + 일관 |
| controller | memory, cpu, blkio, devices, ... | 같음 (+ 일부 통합) |
| Linux 5.x+ 기본 | mixed | v2 단독 (대부분 distro) |

→ 신규 시스템 = **v2**. 옛 K8s / Docker = v1 가능.

---

## 3. 확인

```bash
mount | grep cgroup
# cgroup2 on /sys/fs/cgroup type cgroup2

cat /sys/fs/cgroup/cgroup.controllers
# cpu io memory pids ...

# systemd 의 cgroup 사용 여부
systemctl --version | head -1
systemd-cgls                       # cgroup 트리 시각화
systemd-cgtop                       # top 비슷
```

---

## 4. 기본 조작 (v2)

```bash
# 새 cgroup 생성
mkdir /sys/fs/cgroup/myapp

# controller 활성
echo "+cpu +memory +io" > /sys/fs/cgroup/cgroup.subtree_control

# 프로세스 추가
echo $PID > /sys/fs/cgroup/myapp/cgroup.procs

# 제한 설정
echo "200000 1000000" > /sys/fs/cgroup/myapp/cpu.max     # 20% (200ms / 1000ms)
echo "1G"               > /sys/fs/cgroup/myapp/memory.max
echo "200M"             > /sys/fs/cgroup/myapp/memory.high

# 통계
cat /sys/fs/cgroup/myapp/cpu.stat
cat /sys/fs/cgroup/myapp/memory.current
```

---

## 5. 주요 Controller

### 5.1 cpu

```
cpu.max     "max" 또는 "quota period" (us)
cpu.weight  1-10000 (기본 100)
cpu.stat    usage_usec, nr_periods, nr_throttled, throttled_usec
```

자세히 → [[../scheduling/cfs#12-cfs-bandwidth-control]]

### 5.2 memory

```
memory.max     hard limit (초과 시 OOM)
memory.high    soft limit (reclaim 우선)
memory.low     보호 (가능하면 reclaim X)
memory.swap.max
memory.current
memory.stat    rss / cache / pgfault / ...
memory.events  oom / oom_kill / max
memory.pressure  PSI
```

### 5.3 io

```
io.max         8:0 rbps=100M wbps=100M riops=1000 wiops=1000
io.weight       100
io.stat         읽기/쓰기 byte / ms
```

### 5.4 pids

```
pids.max       1024
pids.current
```

→ fork bomb 보호.

### 5.5 cpuset

```
cpuset.cpus    0-3
cpuset.mems    0
```

→ pinning.

### 5.6 hugetlb / rdma / misc
특수.

---

## 6. systemd 와 cgroup

systemd 가 모든 서비스 / 사용자 세션을 cgroup 으로 관리:

```bash
systemd-cgls
# /
# ├─ system.slice
# │   ├─ nginx.service
# │   ├─ mysql.service
# ├─ user.slice
# │   └─ user-1000.slice
# │       └─ session-1.scope

systemctl status nginx       # cgroup 정보 포함
systemctl set-property nginx CPUQuota=50% MemoryMax=2G
```

`/etc/systemd/system/myapp.service`:
```ini
[Service]
ExecStart=/usr/bin/myapp
MemoryMax=4G
CPUQuota=200%               # 2 코어
TasksMax=1024
IOWeight=200
```

---

## 7. Docker / Kubernetes

### 7.1 Docker

```bash
docker run --cpus=2 --memory=4g --pids-limit=1024 ubuntu

# 내부 cgroup
docker inspect <c> | grep -i cgroup
ls /sys/fs/cgroup/system.slice/docker-<id>.scope/
```

### 7.2 K8s

```yaml
resources:
  requests:
    cpu: "500m"        # 0.5 코어
    memory: "256Mi"
  limits:
    cpu: "2"
    memory: "4Gi"
```

→ kubelet 가 cgroup 으로 변환.

---

## 8. CPU throttling — 함정

```
limit 1 CPU + burst 워크로드
→ short 100ms 동안 100% 사용 → 다음 100ms throttle
→ tail latency 폭증
```

`cpu.stat` 의 `nr_throttled` / `throttled_usec` 으로 진단.

해결:
- limit 늘림 (또는 제거)
- burst 줄임
- `cpu.max_burst` (5.14+) 로 일시 초과 허용

자세히 → [[../scheduling/cfs#12-cfs-bandwidth-control]]

---

## 9. memory.high vs memory.max

```
memory.high — soft. 초과 시 reclaim 적극 + throttle
memory.max  — hard. 초과 시 OOM
```

K8s 는 memory.max 만 — burst 시 즉시 OOM. memory.high 활용 권장 (kubelet 옵션).

---

## 10. OOM in cgroup

```
cgroup 안에서 RSS > memory.max → 그 cgroup 안의 task OOM Killer
→ 호스트 OOM 보호
```

```bash
dmesg | grep -i 'oom'
cat /sys/fs/cgroup/myapp/memory.events
# oom 1
# oom_kill 1
```

자세히 → [[../memory/swap-oom]]

---

## 11. PSI — Pressure Stall Information

```bash
cat /sys/fs/cgroup/myapp/cpu.pressure
cat /sys/fs/cgroup/myapp/memory.pressure
cat /sys/fs/cgroup/myapp/io.pressure
# some avg10=12.34 avg60=8.0 avg300=4.5 total=...
# full avg10=...
```

- `some` — 일부 task stall
- `full` — 모든 task stall

→ systemd-oomd 가 사용. 조기 개입.

---

## 12. delegate

systemd 의 delegate:
```ini
Delegate=yes
```

→ container 안에서 cgroup 자체 관리 가능.

---

## 13. 운영 모니터링

```bash
systemd-cgtop                    # cgroup 별 top
cgtop                             # libcgroup
docker stats
kubectl top pod / node

# 직접
cat /sys/fs/cgroup/<group>/cpu.stat
cat /sys/fs/cgroup/<group>/memory.current
```

Prometheus cAdvisor 가 표준 exporter.

---

## 14. v1 의 잔재

```bash
mount | grep cgroup
# cgroup on /sys/fs/cgroup/cpu type cgroup (rw,cpu)
# cgroup on /sys/fs/cgroup/memory type cgroup (rw,memory)
# ...
```

각 controller 별 디렉토리. 일관성 ↓ → v2 로 이전.

```ini
# /etc/default/grub
GRUB_CMDLINE_LINUX="systemd.unified_cgroup_hierarchy=1"
```

대부분 modern distro = v2.

---

## 15. 함정

### 15.1 CPU limit 의 throttle
burst 워크로드 = 큰 latency spike.

### 15.2 memory.max + JVM
heap + native + GC overhead 고려.

### 15.3 OOM 후 응용 죽음 무시
`memory.events.oom_kill` 모니터링.

### 15.4 cgroup v1 + 새 docker
일부 기능 (cpu.weight) v2 만.

### 15.5 nested cgroup
parent 제한이 children 의 합 ≥ 일 때만 의미.

### 15.6 controller 활성 깜빡
`cgroup.subtree_control` 에 +controller 안 하면 children 에서 사용 X.

### 15.7 K8s 의 burstable / guaranteed
QoS 클래스 (Guaranteed / Burstable / BestEffort) → 다른 cgroup 적용.

### 15.8 systemd 가 모든 process 의 cgroup parent
직접 cgroup 만들면 systemd 가 reset 가능. delegate.

---

## 16. 학습 자료

- `Documentation/admin-guide/cgroup-v2.rst`
- **systemd.resource-control(5)**
- **Container Performance Analysis** — Brendan Gregg
- **Resource Management Guide** — Red Hat

---

## 17. 관련

- [[namespace]]
- [[container]]
- [[../scheduling/cfs]]
- [[../memory/swap-oom]]
- [[virtualization]] — Virt hub
