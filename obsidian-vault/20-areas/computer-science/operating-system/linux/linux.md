---
title: "Linux (Hub)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T16:00:00+09:00
tags:
  - operating-system
  - linux
  - hub
---

# Linux (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | hub + 15 개 세부 노트 |

**[[../operating-system|↑ OS hub]]**

---

## 1. 한 줄

1991 Linus Torvalds 의 monolithic kernel. UNIX 호환 + 오픈소스 → 현재 인터넷 / 클라우드 / Android / 임베디드의 표준.

---

## 2. Kernel + 배포판

```
Linux Kernel (Linus)
  + GNU userland (glibc, coreutils, ...)
  + systemd / init
  + 배포판 (Ubuntu, Debian, RHEL, Arch, ...)
  + 응용 / 패키지 관리
= "Linux 시스템"
```

자세히 → [[distributions]], [[kernel-architecture]]

---

## 3. 부팅 흐름

```
BIOS/UEFI → bootloader (GRUB) → kernel + initramfs
→ /sbin/init (systemd)
→ targets / services
→ login
```

자세히 → [[boot-process]]

---

## 4. 핵심 실무 영역

| 영역 | 노트 |
| --- | --- |
| 배포판 / 패키지 | [[distributions]], [[package-managers]] |
| 커널 아키텍처 | [[kernel-architecture]] |
| 부팅 | [[boot-process]] |
| systemd | [[systemd]] |
| Shell / 명령 | [[shell-commands]] |
| 권한 / 파일 | [[file-permissions]] |
| 네트워크 운영 | [[networking]] |
| 모니터링 | [[monitoring]] |
| 로깅 | [[logging]] |
| Cron / Timer | [[cron-systemd-timer]] |
| SSH | [[ssh]] |
| /proc / /sys | [[proc-sys]] |
| strace / perf | [[strace-perf]] |
| eBPF | [[ebpf]] |
| 튜닝 | [[tuning]] |

---

## 5. Linux 커널의 특징

- **monolithic** — 대부분 기능이 kernel 안 (vs microkernel)
- **modular** — module 동적 로드 (`lsmod`, `insmod`)
- **preemptive** + RT 옵션
- **multi-platform** (x86/ARM/RISC-V/PowerPC/...)
- **VFS, networking, scheduling, MM, IPC, security** 모두 자체

---

## 6. 거대한 ecosystem

```
응용 — millions (open source)
패키지 매니저 — apt, dnf, pacman, ...
컨테이너 / k8s / 클라우드의 기반
Android (Linux 변형)
임베디드 (Yocto, Buildroot, OpenWrt, ...)
HPC / supercomputers (top500 의 100%)
```

---

## 7. 실무 자주 쓰는 도구 한눈에

```bash
# 시스템 정보
uname -a
lsb_release -a
cat /etc/os-release
hostnamectl

# 프로세스
ps aux
top / htop / btop
pidstat
pgrep / pkill

# 메모리 / I/O
free -h
vmstat 1
iostat -x 1
sar -A

# 네트워크
ip a / ip r
ss -tln / ss -tunap
ping / mtr / traceroute / tracepath
curl / wget
tcpdump / wireshark

# 파일
ls / find / locate
df -h / du -sh
mount / umount / lsblk
tar / zip / gzip / xz / zstd

# 패키지
apt / dnf / pacman / snap / flatpak

# systemd
systemctl / journalctl / loginctl

# 로그
journalctl -u service -f
tail -f /var/log/...
dmesg

# 트레이싱
strace / perf / bpftrace / dtrace
```

자세히는 각 노트에.

---

## 8. 면접 핵심

1. **fork / exec / wait** 의 흐름.
2. **/proc/$PID/...** 파일 구조.
3. **systemd unit** 구조 + Type / Restart.
4. **cgroups v1 vs v2**.
5. **ulimit / sysctl** 의 차이.
6. **iptables / nftables**.
7. **journalctl** vs syslog.
8. **strace / perf / eBPF** 의 용도.
9. **dpkg / apt** vs **rpm / dnf**.
10. **shell 의 환경변수 / IFS / quoting**.

---

## 9. 학습 자료

- **Linux Documentation Project** — tldp.org
- **The Linux Programming Interface** — Kerrisk (필독)
- **UNIX and Linux System Administration Handbook**
- **brendangregg.com** — perf
- **lwn.net** — kernel 최신
- **man pages** — `man -k` / `man 7 ...`

---

## 10. 관련

- [[../operating-system|↑ OS hub]]
- [[../../network/network]]
- [[../virtualization/container]]
