---
title: "Linux Kernel 아키텍처"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T16:10:00+09:00
tags:
  - operating-system
  - linux
  - kernel
---

# Linux Kernel 아키텍처

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 서브시스템 구조 |

**[[linux|↑ Linux hub]]**

---

## 1. 한 줄

monolithic 커널 — 거의 모든 OS 기능이 한 process (kernel space) 안. module 로 동적 확장.

크기: ~30M+ 줄 (2025), 7000+ contributor.

---

## 2. 서브시스템

```
┌──────────────────────────────────────┐
│      System Call Interface           │
├──────────────────────────────────────┤
│  Process │ Memory │ VFS │ Network │ ... │
│   Mgmt   │  Mgmt  │     │  Stack   │
├──────────────────────────────────────┤
│      Architecture-specific            │
│  (x86, ARM, RISC-V, ...)             │
├──────────────────────────────────────┤
│      Device Drivers                   │
└──────────────────────────────────────┘
       ↑ Hardware
```

| Subsystem | 디렉토리 (kernel source) |
| --- | --- |
| Process / sched | `kernel/`, `kernel/sched/` |
| Memory mgmt | `mm/` |
| VFS / Filesystem | `fs/` |
| Network | `net/` |
| Block layer | `block/` |
| Drivers | `drivers/` |
| Crypto | `crypto/` |
| Security | `security/` |
| Arch | `arch/x86`, `arch/arm64`, ... |
| ipc | `ipc/` |
| init | `init/` |
| lib | `lib/` |

---

## 3. monolithic vs microkernel

| | Monolithic (Linux) | Microkernel (Mach, L4, seL4) |
| --- | --- | --- |
| 위치 | 모두 kernel | 최소 — 메시지 + 스케줄 + IPC |
| 성능 | 빠름 (호출 X) | 느림 (IPC) |
| 안정성 | 한 부분 죽으면 panic | 더 격리 |
| 예 | Linux, *BSD | macOS XNU (hybrid), QNX |

Linux = monolithic + module (혼합 hybrid 라고도).

---

## 4. Module

```bash
lsmod
modprobe nvidia
modprobe -r nvidia
modinfo nvidia
dmesg | grep nvidia

# /lib/modules/$(uname -r)/
ls /lib/modules/$(uname -r)/kernel
```

- 동적 load/unload
- 디바이스 드라이버 / 파일시스템 / 알고리즘
- in-tree / out-of-tree (DKMS 로 build)

---

## 5. /proc 과 /sys

### 5.1 /proc
프로세스 + 시스템 상태 (procfs).

### 5.2 /sys
디바이스 / 커널 객체 (sysfs).

자세히 → [[proc-sys]]

---

## 6. 디바이스 모델

```
sysfs:
  /sys/bus/pci/devices/...
  /sys/class/net/eth0/
  /sys/block/sda/

udev:
  /etc/udev/rules.d/
  새 디바이스 → kernel uevent → udev → /dev/ 생성
```

```bash
udevadm info /dev/sda
udevadm monitor
```

---

## 7. Interrupts / IRQ

```bash
cat /proc/interrupts
# CPU0  CPU1  ...
# 19:   123    456    IO-APIC  19-fasteoi  eth0

# IRQ 분산
cat /proc/irq/19/smp_affinity
# 자동: irqbalance daemon
```

- hardirq — top half (빠르게 처리)
- softirq / tasklet / workqueue — bottom half

NIC 의 RPS / RFS / XDP 도 IRQ 관련.

---

## 8. /proc/kallsyms

```bash
sudo cat /proc/kallsyms | head
# ffffffff81000000 T _stext
# ffffffff81000000 T startup_64
# ...
```

kernel 의 모든 symbol — KASLR 활성 시 root 만.

eBPF / kprobe 의 토대.

---

## 9. 버전 / 빌드

```bash
uname -r                  # 6.5.0-15-generic
cat /proc/version
```

명명:
```
major.minor.patch[-extraversion]
6.5.0-15-generic
```

LTS: 5.4, 5.10, 5.15, 6.1, 6.6, ...

---

## 10. KConfig — Kernel Config

```bash
# 현재 커널의 config
cat /boot/config-$(uname -r) | grep CONFIG_PREEMPT

# 또는
zcat /proc/config.gz
```

`CONFIG_*` flags 로 기능 선택. 빌드 시 결정.

```bash
# 커널 소스에서
make menuconfig
make -j$(nproc)
sudo make modules_install install
```

---

## 11. printk / dmesg

```bash
dmesg
dmesg -w                # follow
dmesg -T                 # timestamp
journalctl -k --since "1 hour ago"
```

kernel 의 ring buffer (`/dev/kmsg`).

---

## 12. panic / oops

```
panic — unrecoverable, system halt
oops  — recoverable, process killed
```

```bash
# 후속 reboot
sysctl kernel.panic=10           # 10s 후 reboot
```

panic 시:
- kdump 로 vmcore 저장
- crash 분석

---

## 13. RCU (Read-Copy-Update)

kernel 의 lock-free 동시성 mechanism.

```
reader 는 락 없이 read
writer 는 새 버전 만들고 atomic 으로 pointer swap
옛 버전은 모든 reader 끝난 후 free
```

→ kernel 의 read-heavy 자료구조 (route table, ...).

자세히 → [[../synchronization/rwlock#10-rcu-read-copy-update]]

---

## 14. Workqueue / kthread

```bash
ps auxf | grep '\['
# [kworker/0:0]
# [ksoftirqd/0]
# [migration/0]
# [rcu_sched]
# [oom_reaper]
# ...
```

`[name]` = kernel thread. softirq / 백그라운드 작업.

---

## 15. eBPF — kernel programmability

kernel 안에서 안전하게 사용자 정의 program 실행.

자세히 → [[ebpf]]

```bash
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s\n", str(args->filename)); }'
```

---

## 16. 보안 — LSM

```
SELinux, AppArmor, Smack, TOMOYO, Yama, Landlock, ...
kernel 의 hook 에 보안 정책
```

자세히 → [[../security/selinux-apparmor]]

---

## 17. Network Stack

```
Application (socket API)
  ↓
INET (TCP, UDP, ICMP)
  ↓
IP (IPv4 / IPv6)
  ↓
Netfilter (iptables / nftables)
  ↓
Network Driver (NIC)
```

자세히 → [[../../network/network]]

---

## 18. Block Layer

```
응용 → File System → Block layer (request queue) → driver → device
```

- multi-queue (blk-mq, 5.0+ default)
- I/O scheduler (mq-deadline, kyber, bfq)
- io_uring 통합

---

## 19. 컨테이너 / 가상화 kernel feature

- namespace (8 종)
- cgroups (v2)
- seccomp
- capabilities
- LSM hooks
- veth / bridge / vxlan
- overlay fs
- KVM (in-kernel hypervisor)

자세히 → [[../virtualization/virtualization]]

---

## 20. 함정

### 20.1 LTS vs latest
운영 = LTS (5.15, 6.1, 6.6). 새 기능 (io_uring 등) 은 버전 확인.

### 20.2 out-of-tree module
DKMS 필요. 커널 upgrade 시 rebuild.

### 20.3 KASLR + kallsyms
보안 + 디버그 trade-off.

### 20.4 sysctl 의 영구화
`/etc/sysctl.conf` 또는 `/etc/sysctl.d/`.

### 20.5 dmesg 가 비어있음
journalctl -k 또는 권한.

### 20.6 oops 후 unstable
빠른 재부팅 권장.

---

## 21. 학습 자료

- **Linux Kernel Development** — Robert Love
- **Understanding the Linux Kernel** — Bovet/Cesati
- **The Linux Programming Interface** — Kerrisk
- **kernel.org** Documentation/
- **LWN.net** — 최신 변화

---

## 22. 관련

- [[boot-process]]
- [[proc-sys]]
- [[ebpf]]
- [[../operating-system|↑ OS hub]]
- [[linux]] — Linux hub
