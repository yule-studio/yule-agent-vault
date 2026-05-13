---
title: "Linux vs macOS vs Windows — 비교"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:45:00+09:00
tags:
  - operating-system
  - topic
  - comparison
---

# Linux vs macOS vs Windows

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 3 OS 비교 |

**[[topics|↑ Topics hub]]**

---

## 1. 한눈에

| | Linux | macOS | Windows |
| --- | --- | --- | --- |
| Kernel | Monolithic (Linux) | XNU (Mach + BSD) | NT (hybrid) |
| Userland | GNU + BSD 변형 | BSD (변형) | Win32 + .NET |
| 라이선스 | GPL / 오픈소스 | Proprietary | Proprietary |
| 데스크탑 점유 | 4-5% | 15-20% | 70%+ |
| 서버 점유 | 80%+ | 0% | 20% (downward) |
| 모바일 (Android = Linux) | 70%+ | iOS 30% | 1% |
| 임베디드 | 거의 100% | 0 | 일부 |
| 클라우드 | 95%+ | 0 | 일부 |

---

## 2. Kernel 비교

### 2.1 Linux
- Monolithic + module
- 35M+ lines
- 7000+ contributors
- 모든 architecture (x86, ARM, RISC-V, PowerPC, ...)

### 2.2 macOS XNU
- Hybrid (Mach microkernel core + BSD layer)
- iOS / iPadOS / tvOS / watchOS 공통

### 2.3 Windows NT
- Hybrid
- Dave Cutler design (DEC VMS 영감)
- Subsystem (Win32, POSIX 옛, WSL)

자세히:
- [[../linux/kernel-architecture]]
- [[../macos/macos]]
- [[../windows/nt-architecture]]

---

## 3. PID 1 / Init

| | |
| --- | --- |
| Linux | systemd (대부분 distro) |
| macOS | launchd |
| Windows | smss.exe / wininit / services.exe |

기능 비슷:
- 서비스 관리
- 자식 reap
- target / runlevel

---

## 4. Shell

| | 기본 |
| --- | --- |
| Linux | bash / dash / zsh |
| macOS | zsh (Catalina+) / bash |
| Windows | PowerShell / cmd / WSL bash |

---

## 5. 파일 시스템

| | 기본 |
| --- | --- |
| Linux | ext4 / XFS / Btrfs / ZFS / F2FS |
| macOS | APFS (2017+) / 옛 HFS+ |
| Windows | NTFS / ReFS (서버) / FAT32 |

자세히 → [[../filesystem/ext-xfs-zfs]]

---

## 6. 권한 모델

| | |
| --- | --- |
| Linux | UID/GID + mode + ACL + Capability + SELinux/AppArmor |
| macOS | UNIX + ACL + TCC + Gatekeeper + SIP |
| Windows | SID + DACL + Token + UAC + Integrity Level |

자세히 → [[../security/permissions]]

---

## 7. 패키지

| | |
| --- | --- |
| Linux | apt / dnf / pacman / apk / snap / flatpak |
| macOS | Homebrew / MacPorts |
| Windows | winget / Chocolatey / Scoop |

---

## 8. 컨테이너 / 가상화

| | Native |
| --- | --- |
| Linux | namespace + cgroup → Docker 직접 |
| macOS | Linux VM 안 (Docker Desktop / Colima) |
| Windows | Hyper-V + WSL2 + Windows Container |

→ Linux 가 컨테이너 native. mac / Windows = VM 통해.

자세히 → [[../virtualization/container]]

---

## 9. IPC

| | |
| --- | --- |
| Linux | pipe / Unix socket / shm / mq / signal / D-Bus |
| macOS | + Mach port + XPC |
| Windows | + Named pipe + COM/RPC + ALPC |

---

## 10. I/O 모델

| | |
| --- | --- |
| Linux | epoll / io_uring / AIO |
| macOS | kqueue |
| Windows | IOCP |

자세히 → [[../io/io-models]]

---

## 11. 디버그 도구

| 영역 | Linux | macOS | Windows |
| --- | --- | --- | --- |
| process | ps / top | ps / top | tasklist / procexp |
| trace syscall | strace | dtrace / dtruss | procmon |
| profile | perf | Instruments / sample | WPR / Perf |
| log | journalctl | log show / Console.app | eventvwr |
| package | apt/dnf | brew | winget |

---

## 12. 시스템 콜

| Linux | macOS | Windows |
| --- | --- | --- |
| ~400 | ~500 (Mach + BSD) | Nt* + Zw* 수천 |
| POSIX 표준 | POSIX 호환 | Win32 API (POSIX subsystem 옛) |
| 안정 ABI | 변화 가능 (private) | 안정 |

Linux = syscall stable. macOS = `libSystem` 통해서만 권장.

---

## 13. 보안 모델

| | |
| --- | --- |
| Linux | DAC + Capability + LSM (SELinux/AppArmor) + seccomp |
| macOS | TCC + SIP + Gatekeeper + sandbox + entitlement |
| Windows | UAC + DACL + Token + AppContainer + Defender |

iOS / Android = 가장 엄격. macOS / Windows = consumer 친화.

자세히 → [[../security/security]]

---

## 14. 개발 환경

| | Linux | macOS | Windows |
| --- | --- | --- | --- |
| Native dev | 직접 | 직접 + brew | WSL2 + native |
| C/C++ | GCC / Clang | Clang | MSVC + Clang |
| Docker | native | VM | WSL2 / VM |
| Python / Node | apt + venv | brew + pyenv | python.org + nvm |
| Java / Go / Rust | sdkman / native | brew | choco / winget |

대부분 backend dev = macOS 또는 WSL2.
Linux server 가 production target → 개발도 Linux-like.

---

## 15. 데스크탑 GUI

| | |
| --- | --- |
| Linux | X11 / Wayland + GNOME / KDE / XFCE |
| macOS | Aqua (Cocoa) |
| Windows | DWM (Composited) |

UNIX 의 X11 → Wayland 마이그.

---

## 16. 응용 시나리오

| 시나리오 | OS |
| --- | --- |
| 서버 / 클라우드 | Linux |
| 컨테이너 host | Linux |
| 데스크탑 (디자인) | macOS |
| 데스크탑 (게임) | Windows / Linux (Steam) |
| 데스크탑 (사무) | Windows |
| 데스크탑 (dev) | macOS / Linux / Windows+WSL |
| 임베디드 | Linux / 임베디드 RTOS |
| 모바일 | Android (Linux) / iOS (macOS 계열) |
| 슈퍼컴 | Linux (top500 100%) |
| AI / ML training | Linux |
| 금융 / HFT | Linux RT |
| 우주 / 항공 | RTOS (VxWorks, QNX) |

---

## 17. 클라우드 사실상 Linux

```
AWS / GCP / Azure 의 host = Linux
이 위에 KVM / Hyper-V 로 guest VM
guest = Linux (90%+) / Windows (15%)
```

→ Linux 가 인프라의 표준.

---

## 18. 면접에서 자주

### 18.1 "왜 Linux 가 서버 표준?"
- 오픈소스 + 라이선스 자유
- 거대 ecosystem
- 안정 + 보안
- 모든 hardware
- container / kubernetes native

### 18.2 "macOS 가 UNIX 인가?"
- BSD 기반 → UNIX 호환 (Linux 는 UNIX-like)
- POSIX 인증 (Apple)
- 단, 폐쇄적

### 18.3 "WSL2 의 의미"
- Windows 에서 Linux dev
- Hyper-V VM 안의 진짜 Linux kernel
- file system 통합
- Docker / VS Code 와 자연스러운 통합

---

## 19. 함정

### 19.1 "POSIX 호환"
- Linux, macOS = 대부분 호환
- 단, 세부 (system call argument / errno / option) 다름
- WSL1 = 옛 translation, WSL2 = 진짜 Linux

### 19.2 GNU coreutils vs BSD
- ls / sed / date / xargs 의 옵션 차이
- mac script → Linux 에서 fail 흔함

### 19.3 line ending
- Linux/Mac = LF (\n)
- Windows = CRLF (\r\n)
- git autocrlf

### 19.4 path separator
- Linux/Mac = `/`
- Windows = `\` 또는 `/`

---

## 20. 학습 자료

- **Linux Kernel Development** — Robert Love
- **macOS Internals** — Jonathan Levin
- **Windows Internals** — Mark Russinovich
- **The Design of the UNIX Operating System** — Bach

---

## 21. 관련

- [[../linux/linux]]
- [[../macos/macos]]
- [[../windows/windows]]
- [[topics]] — Topics hub
