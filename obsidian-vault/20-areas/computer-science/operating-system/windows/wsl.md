---
title: "WSL — Windows Subsystem for Linux"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T17:50:00+09:00
tags:
  - operating-system
  - windows
  - wsl
---

# WSL

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | WSL 1 vs 2 + 운영 |

**[[windows|↑ Windows hub]]**

---

## 1. 한 줄

Windows 안에서 Linux 환경 실행. 개발자 친화 + Docker / VS Code 표준.

---

## 2. WSL1 vs WSL2

| | WSL1 | WSL2 |
| --- | --- | --- |
| 방식 | syscall translation | Hyper-V VM + 진짜 Linux kernel |
| Linux 호환 | 부분 (일부 syscall X) | 완전 |
| Linux fs I/O | 같은 NTFS — 어색 | 가상 disk (ext4) — 빠름 |
| Windows fs (`/mnt/c`) | 같은 disk — 빠름 | 9P protocol — 느림 |
| Docker | 어려움 | 표준 |
| 시작 시간 | 즉시 | 1-2 초 |
| 메모리 | 일반 process | VM 메모리 |

→ **WSL2 권장**. Microsoft 도 WSL2 가 표준.

---

## 3. 설치

```powershell
# 가장 단순 (관리자)
wsl --install
# 자동 활성 + Ubuntu 설치

# 특정 배포
wsl --install -d Ubuntu-22.04
wsl --install -d Debian
wsl --list --online                       # 가능한 배포 목록

# 옛 방식 (옵션 활성)
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

---

## 4. 기본 명령

```powershell
wsl --list
wsl --list --verbose
wsl --list --running

wsl                                       # default 진입
wsl -d Ubuntu                              # 특정 배포
wsl -d Ubuntu -u root                       # 다른 user
wsl --shutdown                             # 모두 종료
wsl --terminate Ubuntu

wsl --set-default Ubuntu
wsl --set-default-version 2
wsl --set-version Ubuntu 2
wsl --update
```

---

## 5. Export / Import

```powershell
# backup / migration
wsl --export Ubuntu C:\backups\ubuntu.tar
wsl --unregister Ubuntu

wsl --import Ubuntu2 C:\WSL\Ubuntu2 C:\backups\ubuntu.tar
wsl --import-in-place CustomDistro C:\path\to\ext4.vhdx
```

→ 가벼운 dev VM. 여러 환경 분리.

---

## 6. Distro 등록 / 사용자 정의

```bash
# 첫 접속 시 username / password 설정

# default user 변경
# Windows 11+ 에선 자동, 옛 버전:
# /etc/wsl.conf:
[user]
default=alice
```

---

## 7. /etc/wsl.conf

```ini
[boot]
systemd=true                              # systemd 활성 (WSL2 0.67+)

[network]
hostname=mywsl
generateHosts=true
generateResolvConf=true

[interop]
enabled=true
appendWindowsPath=true                    # PATH 에 Windows binary

[user]
default=alice

[automount]
enabled=true
options="metadata,umask=22,fmask=11"
mountFsTab=true

[wsl2]
memory=8GB                                  # VM 메모리 한계
processors=4
swap=4GB
localhostForwarding=true
```

또는 `~/.wslconfig` (모든 distro):

```ini
[wsl2]
memory=16GB
processors=8
```

---

## 8. systemd in WSL2

WSL 0.67+ (Windows 11) — systemd 지원.

```bash
# /etc/wsl.conf
[boot]
systemd=true

# wsl --shutdown 후 재시작
systemctl list-units
```

→ 일반 Linux 처럼 service 관리.

---

## 9. 파일 시스템

```
Windows fs from WSL:
  /mnt/c/Users/me/        — slow, NTFS 그대로

WSL Linux fs from Windows:
  \\wsl$\Ubuntu\home\me   — 또는 \\wsl.localhost\
```

권장:
- 프로젝트 = WSL fs (Linux 안 `~`)
- Windows 와 공유 필요한 파일만 `/mnt/c`

---

## 10. Network

```
WSL2 = NAT 안의 VM
Windows host ↔ WSL2 = `localhost` 또는 `mirror` 모드 (Windows 11+)
```

```ini
# .wslconfig
[wsl2]
networkingMode=mirrored                   # Windows 의 NIC 그대로
dnsTunneling=true
firewall=true
```

→ Windows 11+ 기본 mirror 모드 — VPN / 회사망 호환 ↑.

---

## 11. GUI 응용 — WSLg

WSL 1.0+ (Windows 11) — GUI Linux 응용 직접:

```bash
sudo apt install gedit
gedit                                      # GUI 창 뜸
```

X11 / Wayland 자동. WSLg.

---

## 12. GPU

```bash
nvidia-smi                                 # WSL2 안에서 GPU
```

- CUDA 가능
- DirectX 12 driver share
- AI / ML / 게임 개발에 강력

---

## 13. Docker Desktop / Linux Docker

### 13.1 Docker Desktop
WSL2 backend. Windows 의 Docker = WSL2 안의 Linux Docker.

### 13.2 Native Docker in WSL

```bash
# WSL2 안에서 직접
sudo apt install docker.io
sudo service docker start
# 또는 systemd 활성 후
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
```

Docker Desktop 없이 — 라이선스 / 가벼움.

---

## 14. VS Code Remote

```
Windows VS Code + WSL extension
→ \\wsl$\Ubuntu\path\project 열기
→ extension / language server 가 Linux 안에서
```

→ Linux 개발의 표준 방식 (Windows 측).

---

## 15. 자주 보는 작업

```bash
# Linux 패키지
sudo apt update && sudo apt upgrade -y
sudo apt install build-essential git python3

# Node / Python 등 dev
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install nodejs

# Windows 와 통합
explorer.exe .                             # 현재 디렉토리 탐색기
notepad.exe file.txt
code .                                      # VS Code (Windows host)

# WSL 안에서 Windows path
echo $PATH                                  # /mnt/c/Windows/... 포함
```

---

## 16. 진단

```bash
# WSL 버전 / kernel
uname -a
cat /proc/version
# Linux 5.15.x-microsoft-standard-WSL2

wsl --version       # (PowerShell)
```

---

## 17. 함정

### 17.1 `/mnt/c` 위에서 build
crossing NTFS ↔ ext4 — 매우 느림. Linux fs 권장.

### 17.2 line ending
Windows = CRLF, Linux = LF.
```bash
git config --global core.autocrlf input
```

### 17.3 systemd 누락
옛 WSL → service 시작 어려움. `wsl.conf` 의 `systemd=true`.

### 17.4 메모리 폭증
WSL2 VM 의 메모리 회수 lazy. `.wslconfig` 의 limit + `wsl --shutdown`.

### 17.5 network drift
sleep / VPN 후 DNS 깨짐. `wsl --shutdown` 후 재시작.

### 17.6 disk image (ext4.vhdx) 폭증
delete 후에도 줄지 않음. `wsl --shutdown` + Optimize-VHD (Hyper-V).

### 17.7 docker.sock 권한
WSL 안에서 `sudo docker` 또는 group 추가.

### 17.8 cross-fs symlink
Windows fs 의 symlink ≠ Linux symlink. NTFS junction.

---

## 18. WSL 1 → 2 마이그

```powershell
wsl --set-version Ubuntu 2
```

- Hyper-V 활성 필요
- 큰 디스크면 시간 ↑

---

## 19. WSL 사용 시나리오

- **Linux 개발 환경** — 표준
- **Docker** — Linux container 직접
- **CI 로컬 테스트** — Linux 시나리오
- **DevOps tool** — kubectl, terraform, ansible
- **AI / ML** — CUDA + Python

---

## 20. 학습 자료

- **Microsoft WSL Documentation** — learn.microsoft.com/wsl
- **WSL Github** — github.com/microsoft/WSL
- **Awesome WSL**

---

## 21. 관련

- [[windows]] — Windows hub
- [[../linux/linux]] — Linux 일반
- [[../virtualization/container]] — Docker
