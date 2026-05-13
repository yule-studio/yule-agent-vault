---
title: "NT Architecture — Windows Kernel"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T17:40:00+09:00
tags:
  - operating-system
  - windows
  - kernel
---

# NT Architecture

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | NT 커널 / object / IRQL |

**[[windows|↑ Windows hub]]**

---

## 1. 한 줄

Dave Cutler 가 1993 부터 설계. Mach + VMS 영감의 **hybrid kernel**.

```
User Mode:
  Application
  Subsystem (Win32, POSIX, WSL)
  ntdll.dll (Native API wrapper)

Kernel Mode:
  Executive (Object Manager, Process, Memory, Security, ...)
  Kernel (스케줄러, IRQL, dispatcher)
  HAL (Hardware Abstraction Layer)
  Driver
```

---

## 2. 주요 구성

### 2.1 Executive
- Object Manager (모든 자원 = NT object — file / process / event 등)
- Process / Thread / Job Manager
- Memory Manager
- I/O Manager
- Security Reference Monitor (SRM)
- Plug & Play Manager

### 2.2 Kernel
- Thread Dispatcher (스케줄러)
- IRQL 관리
- DPC / APC (deferred call)
- Spinlock

### 2.3 HAL
하드웨어 추상 — driver 가 같은 API.

### 2.4 Driver
Kernel-mode (가장 큰 보안 표면). User-mode driver framework 도 있음.

---

## 3. Object Manager

모든 자원 = NT object. 통일된 handle / 권한 / 참조 카운터.

```
Object Type:
  Process, Thread, File, Section, Event, Mutex, Semaphore,
  Token, Key (registry), Port, Driver, Device, ...

핸들 (HANDLE) = file descriptor 비슷한 정수
```

```cmd
handle -p notepad.exe                    # Sysinternals
```

---

## 4. IRQL (Interrupt Request Level)

```
HIGH_LEVEL          31
POWER_LEVEL         30
IPI_LEVEL           29
CLOCK_LEVEL         28
PROFILE_LEVEL       27
...
DISPATCH_LEVEL      2  ← scheduler
APC_LEVEL           1
PASSIVE_LEVEL       0  ← 일반 thread
```

→ 높은 IRQL 코드는 sleep / page fault X. spinlock 만.

driver 작성 시 핵심 개념.

---

## 5. 프로세스 / 스레드

### 5.1 Process
- 자원 컨테이너 (handle table, virtual memory)
- 최소 1 thread

### 5.2 Thread
- 진짜 실행 단위
- 같은 process 의 thread 가 메모리 공유

### 5.3 Fiber
사용자 공간 스케줄 — 거의 안 씀.

### 5.4 Job
process 그룹 — 자원 제한 가능 (cgroup 비슷).

```c
CreateProcess(...);
CreateThread(...);
WaitForSingleObject(handle, INFINITE);
```

---

## 6. 메모리

- Virtual address space — 64-bit 에서 128 TB (사용자), 같은 양 (커널)
- Page = 4 KB, Large Page = 2 MB / 1 GB
- Working Set + Page File (Linux 의 swap)
- AWE (Address Windowing Extensions) — 옛 32-bit 큰 메모리

```powershell
Get-Counter '\Memory\Available MBytes'
```

---

## 7. I/O

### 7.1 IRP (I/O Request Packet)
모든 I/O = IRP 가 driver stack 위로 전파.

### 7.2 IOCP (I/O Completion Port)
**Proactor** 모델 — Linux 의 io_uring 동등 (먼저 등장).

```c
HANDLE iocp = CreateIoCompletionPort(...);
ReadFile(h, buf, sz, NULL, &overlapped);
GetQueuedCompletionStatus(iocp, ...);
```

→ ASP.NET / IIS / SQL Server 의 토대.

자세히 → [[../io/io-models]]

---

## 8. 시스템 콜 — System Service Dispatcher (SSDT)

```
User:   ReadFile() in kernel32.dll → NtReadFile() in ntdll.dll → syscall
Kernel: SSDT 의 lookup → NtReadFile() 실행 → return
```

Linux 의 syscall 비슷. `syscall` 명령 (또는 옛 `sysenter`).

---

## 9. 보안 — SID / Token / ACL

```
SID (Security Identifier):
  S-1-5-21-...-1001  (사용자)
  S-1-5-32-544       (Administrators)

Token = process 의 권한 set:
  - Primary SID
  - Group SIDs
  - Privileges (SeShutdownPrivilege, ...)
  - Integrity Level (Low, Medium, High, System)

ACE / ACL — 자원의 권한 list:
  - DACL (Discretionary) — 사용자 정의
  - SACL (System) — audit
```

```cmd
whoami /all
icacls "C:\Path"
```

---

## 10. UAC + Integrity Level

```
System    — kernel / SYSTEM service
High      — admin 권한 (UAC elevation)
Medium    — 일반 user
Low       — sandbox (browser tab)
Untrusted — 가장 격리
```

Internet Explorer / Chrome 의 sandbox 가 Low integrity.

---

## 11. Drivers

- KMDF (Kernel-Mode Driver Framework)
- UMDF (User-Mode Driver Framework) — 안전
- WHQL signing 필수 (보안)
- WDM — 옛 NT driver model

```cmd
driverquery /v
pnputil /enum-drivers
```

---

## 12. 부팅

```
UEFI → Windows Boot Manager (bootmgfw.efi)
  → winload.efi → ntoskrnl.exe + HAL + boot drivers
  → SMSS (Session Manager Subsystem)
    → CSRSS (Win32 subsystem)
    → WININIT
      → SERVICES, LSASS, ...
    → WINLOGON
      → LOGONUI → 사용자 login
```

```cmd
bcdedit
msconfig
```

---

## 13. PE (Portable Executable)

Windows binary 포맷:
- `.exe`, `.dll`, `.sys`
- Linux ELF 동등

`dumpbin /headers myapp.exe` (Visual Studio).

---

## 14. Filesystem

자세히 → [[../filesystem/ext-xfs-zfs#8-ntfs]]

NTFS — journaling + ACL + shadow copy + compression + encryption (EFS).
ReFS (Resilient FS) — 서버용 차세대.

---

## 15. Win32 API 와 .NET

```
Win32 API (C) — 전통
  ↓
.NET (CLR) — 추상화
  ↓ JIT
같은 IL bytecode → CoreCLR (Linux/macOS 도)
```

.NET 8+ = cross-platform.

---

## 16. Subsystem 모델

NT 의 설계 — 여러 subsystem 지원:
- Win32 — 거의 모든 응용
- POSIX (옛 SFU) — 폐기
- OS/2 (옛) — 폐기
- **WSL1** — Linux syscall translation
- **WSL2** — VM 안의 진짜 Linux

→ NT 의 hybrid 설계의 표현.

---

## 17. Sysinternals — 깊은 분석

- **Process Explorer** — 강력한 ps
- **Process Monitor** — strace + ftrace
- **Autoruns** — 시작 항목
- **PsExec** — 원격 명령
- **TCPView** — netstat
- **VMMap** — 메모리 맵
- **WinDbg** — kernel / user debugger

Mark Russinovich (CTO of Azure) 의 도구. Microsoft 무료.

---

## 18. 함정

### 18.1 Kernel-mode driver bug
BSOD (Blue Screen of Death) — kernel panic.

### 18.2 IRQL 위반
DISPATCH_LEVEL 에서 sleep → BSOD.

### 18.3 Handle leak
HANDLE close 안 하면 process 의 handle table 폭증.

### 18.4 ACL 복잡
간단한 chmod 와 달리 — 상속 / 명시 / SID 매핑.

### 18.5 UAC 자동화
admin 권한 자동 요구 어려움 — Task Scheduler / service 설계.

### 18.6 Token vs Process owner
process 가 다른 user 로 실행 (RunAs / service).

### 18.7 WoW64 (32-bit on 64-bit)
별도 registry view (`Wow6432Node`). 디버깅 혼동.

---

## 19. 학습 자료

- **Windows Internals** (Mark Russinovich) — 가장 깊음
- **Microsoft Docs — Kernel-mode Driver Reference**
- **Sysinternals**
- **OSR Online**

---

## 20. 관련

- [[windows]] — Windows hub
- [[powershell]]
- [[wsl]]
