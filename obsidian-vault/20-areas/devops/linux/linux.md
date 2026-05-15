---
title: "Linux — 서버 OS 기본 ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T06:32:00+09:00
tags: [area, devops, linux]
---

# Linux — 서버 OS 기본 ★

**[[../devops|↑ devops]]**

---

## 1. 왜 필요

대부분의 서버 = Linux (CentOS/Ubuntu/Debian/Alpine).  
DevOps 의 모든 도구 (Docker / k8s / nginx) 가 Linux 위에서 동작.

→ **Linux 모르고 DevOps 불가**.

---

## 2. 어떤 distro

| Distro | 용도 | 특징 |
| --- | --- | --- |
| **Ubuntu** | dev / cloud 기본 | apt, 22.04 LTS / 24.04 LTS |
| **Debian** | 안정, 보수적 | apt, Ubuntu 의 base |
| **CentOS Stream / Rocky / Alma** | enterprise (RHEL 호환) | yum/dnf |
| **Alpine** | container (작음 5MB) | musl libc (glibc 호환성 주의) |
| **Amazon Linux** | AWS 최적화 | yum, AWS 통합 |
| **Photon / Bottlerocket** | container host 전용 | 보안 / 작음 |

→ **dev = Ubuntu**, **container = Alpine 또는 distroless**, **AWS = Amazon Linux 또는 Ubuntu**.

---

## 3. 하위 영역

- [[shell-basics]] — 명령 / pipe / redirect / 변수
- [[shell-scripting]] — bash script + best practice
- [[file-permissions]] — chmod / chown / umask / setuid
- [[process-management]] — ps / top / htop / kill / job
- [[systemd]] — service / unit file / journalctl
- [[networking-basics]] — ip / ss / netstat / curl / dig
- [[package-management]] — apt / yum / dnf
- [[users-and-groups]] — useradd / sudo / PAM
- [[ssh]] — key / config / agent / tunnel
- [[performance-troubleshooting]] — CPU/mem/disk/io 분석
- [[pitfalls]] — 흔한 함정

---

## 4. 학습 순서

1. Day 1 — shell-basics + file-permissions
2. Day 2 — process-management + systemd
3. Day 3 — networking-basics + ssh
4. Day 4 — shell-scripting + package-management
5. Day 5 — performance-troubleshooting + pitfalls

---

## 5. 필수 명령 30선 (★)

```bash
# 파일
ls -lh         pwd        cd        cp -r       mv          rm -rf
find / -name   du -sh     df -h     tar -czvf   tar -xzvf

# 텍스트
cat            less       head      tail -f     grep -r     awk        sed
sort           uniq -c    wc -l     cut -f      tr          xargs

# 프로세스
ps aux         top        htop      kill -9     jobs        bg / fg     nohup &

# 네트워크
ip a           ss -tuln   curl -v   wget        dig         nc -zv

# 시스템
uname -a       free -h    uptime    journalctl  systemctl   dmesg
```

---

## 6. 추천 도서

- **The Linux Command Line** (William Shotts, 무료) — beginner
- **Linux System Programming** (Robert Love) — kernel/syscall
- **UNIX Network Programming** (Stevens) — 네트워크
- **Systems Performance** (Brendan Gregg) — performance

---

## 7. 관련

- [[../devops|↑ devops]]
- [[../docker/docker|↗ docker]]
- [[../kubernetes/kubernetes|↗ k8s]]
- [[../nginx/nginx|↗ nginx]]
