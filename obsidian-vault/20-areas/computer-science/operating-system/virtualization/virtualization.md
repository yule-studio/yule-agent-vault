---
title: "Virtualization / Container (Hub)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T14:50:00+09:00
tags:
  - operating-system
  - virtualization
  - container
  - hub
---

# Virtualization / Container (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | hub + 6 개 세부 노트 |

**[[../operating-system|↑ OS hub]]**

---

## 1. 한 줄

**VM** — 하드웨어 추상화, 게스트가 자기 커널.
**컨테이너** — 같은 커널 공유 + namespace/cgroup 격리.

→ 거의 모든 클라우드 인프라의 토대.

---

## 2. 두 영역

| 영역 | 노트 |
| --- | --- |
| VM / Hypervisor | [[vm-hypervisor]] |
| KVM (Linux 의 VM) | [[kvm]] |
| Namespace | [[namespace]] |
| cgroups | [[cgroups]] |
| Container runtime | [[container]] |
| 비교 | [[vm-vs-container]] |

---

## 3. VM (Virtual Machine)

```
Host OS
  ↓
Hypervisor (KVM / VMware / Hyper-V / Xen)
  ↓
Guest VM (자기 커널 + userspace)
```

자세히 → [[vm-hypervisor]]

---

## 4. 컨테이너

```
Host kernel (공유)
  ↓
Namespace (PID / Network / Mount / IPC / UTS / User / Cgroup)
+ cgroup (자원 제한)
  ↓
Container process (자기 rootfs + 격리)
```

자세히 → [[container]]

---

## 5. Namespace

Linux 커널이 제공하는 8 종 격리:

| Namespace | 격리 대상 |
| --- | --- |
| `mnt` | mount point |
| `pid` | PID |
| `net` | network (interface, route, port) |
| `ipc` | SysV IPC, POSIX MQ |
| `uts` | hostname / domainname |
| `user` | UID / GID |
| `cgroup` | cgroup root view |
| `time` (5.6+) | clock |

자세히 → [[namespace]]

---

## 6. cgroups

CPU / memory / I/O / network / device 자원 제한 + 통계.

자세히 → [[cgroups]]

---

## 7. Container Runtime

| Runtime | 역할 |
| --- | --- |
| **runc** | OCI low-level (Docker 의 내부) |
| **containerd** | high-level (Docker, K8s) |
| **CRI-O** | K8s 전용 |
| **podman** | rootless / daemonless |
| **Docker** | 사용자 친화 wrapper |
| **K8s** | orchestration |

자세히 → [[container]]

---

## 8. VM vs Container

| 측면 | VM | Container |
| --- | --- | --- |
| 커널 | 게스트 자기 | 공유 |
| 격리 | 강 | 약 |
| 부팅 | 분 | 초 |
| 크기 | GB | MB |
| 오버헤드 | 큼 | 거의 0 |
| 보안 | 강 (hypervisor) | namespace + seccomp + AppArmor 등 |
| 멀티 OS | ✅ (다른 OS) | 같은 커널만 |

자세히 → [[vm-vs-container]]

---

## 9. 하이브리드 — Lightweight VM

```
Firecracker / Kata Containers / gVisor
```

- VM 의 격리 + 컨테이너의 가벼움
- AWS Lambda / Fargate

자세히 → [[vm-hypervisor#7-light-vm]]

---

## 10. 진화 흐름

```
1990s: VM (VMware)
2000s: KVM (Linux 내장)
2008:  cgroups (Google)
2008:  LXC (Linux Containers)
2013:  Docker (image + UX)
2014:  Kubernetes
2015:  rkt / Open Container Initiative (OCI)
2017:  containerd / Kata / Firecracker
2020+: rootless / Wasm runtime / eBPF 통합
```

---

## 11. 컨테이너 보안 모델

- namespace 격리
- cgroups 자원 제한
- seccomp (syscall filter)
- AppArmor / SELinux (MAC)
- capabilities drop
- read-only rootfs
- user namespace (rootless)

자세히 → [[../security/security]]

---

## 12. 면접 핵심

1. **VM vs 컨테이너** — 격리 / 성능 / 보안.
2. **Namespace** 8 종 + 어떤 격리.
3. **cgroups v1 vs v2**.
4. **Docker 의 내부** — image / overlay / runc.
5. **K8s Pod** = 같은 namespace 공유 컨테이너 그룹.
6. **runc 의 init** — exec / namespace 진입.
7. **rootless container** 의 의미.
8. **gVisor / Kata** — light VM.
9. **OOM in container** — cgroup memory.max.
10. **PID 1 in container** — zombie / signal.

---

## 13. 학습 자료

- **OSTEP** Ch. 53 (virtualization)
- **Linux Kernel Development** + `Documentation/admin-guide/cgroup-v2.rst`
- **Container Security** — Liz Rice
- **runc / containerd / Kubernetes** 소스
- **Brendan Gregg — Container Performance**

---

## 14. 관련

- [[../process/pcb]] — task_struct + nsproxy
- [[../scheduling/cfs]] — cgroup CPU
- [[../security/security]] — sandbox
- [[../operating-system|↑ OS hub]]
