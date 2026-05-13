---
title: "Windows (Hub)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T17:35:00+09:00
tags:
  - operating-system
  - windows
  - hub
---

# Windows (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Windows 의 OS 측면 |

**[[../operating-system|↑ OS hub]]**

---

## 1. 한 줄

Microsoft 의 데스크탑 / 서버 OS. NT 커널 (Dave Cutler, 1993) + Win32 API + UWP.
서버 = Windows Server, 컨테이너 / WSL = Linux 호환.

---

## 2. NT 커널

- Mach 영감의 hybrid (kernel + executive + subsystem)
- HAL (Hardware Abstraction Layer)
- Object Manager (모든 자원 = object)
- IRQL (Interrupt Request Level)
- Process / Thread / Job
- Native API (`Nt*`, `Zw*`) vs Win32 API

NT 6.x = Vista / 7 / 8, NT 10.0 = 10 / 11.

---

## 3. 프로세스 / 스레드

- **Process** = 자원 컨테이너 (메모리 / handle)
- **Thread** = 실행 단위 (Linux 처럼)
- **Fiber** = 사용자 공간 스케줄 (드뭄)
- **Job** = process 그룹 + 제한

```cmd
tasklist
tasklist /v
taskkill /PID 1234
taskkill /IM notepad.exe /F
```

PowerShell:
```powershell
Get-Process
Get-Process -Name nginx
Stop-Process -Name nginx -Force
```

자세히 → [[nt-architecture]]

---

## 4. PowerShell

`cmd.exe` 의 후계. .NET 통합.

```powershell
Get-Process | Where-Object {$_.CPU -gt 100}
Get-Service | Where-Object {$_.Status -eq 'Running'}
Get-Content file.txt
Set-Content -Path file.txt -Value "hello"
Test-Connection google.com

# pipe — object pipeline (text 가 아님)
Get-Process | Sort-Object CPU -Descending | Select-Object -First 5
```

자세히 → [[powershell]]

---

## 5. 서비스

```powershell
Get-Service nginx
Start-Service nginx
Stop-Service nginx
Restart-Service nginx
Set-Service nginx -StartupType Automatic
```

```cmd
sc query nginx
sc start nginx
sc config nginx start= auto
```

GUI = `services.msc`.

---

## 6. Event Log

```powershell
Get-EventLog -LogName Application -Newest 50
Get-WinEvent -LogName System -MaxEvents 50
Get-WinEvent -FilterHashtable @{LogName='Security'; ID=4624}
```

Linux 의 syslog/journal 동등. GUI = `eventvwr.msc`.

---

## 7. 패키지 / 설치

```powershell
# winget (Microsoft 공식)
winget install --id Microsoft.VisualStudioCode
winget list
winget upgrade --all

# Chocolatey
choco install nodejs
choco upgrade all

# Scoop (developer 친화)
scoop install git
```

---

## 8. WSL — Windows Subsystem for Linux

```powershell
wsl --install                          # 자동
wsl --list --online
wsl --install -d Ubuntu

wsl                                     # 진입
wsl --shutdown
wsl --export Ubuntu C:\backup.tar
wsl --import Ubuntu2 C:\WSL\Ubuntu2 C:\backup.tar
```

자세히 → [[wsl]]

---

## 9. NTFS

자세히 → [[../filesystem/ext-xfs-zfs#8-ntfs]]

```cmd
chkdsk C: /F
defrag C:
sfc /scannow

# 권한 (DACL)
icacls "C:\Path"
icacls "C:\Path" /grant Users:F
```

---

## 10. Registry

```cmd
regedit                                # GUI
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion" /v ProductName
reg add ...
reg delete ...
```

Linux 의 /etc 동등. 5 root:
- HKEY_CLASSES_ROOT
- HKEY_CURRENT_USER
- HKEY_LOCAL_MACHINE
- HKEY_USERS
- HKEY_CURRENT_CONFIG

---

## 11. 네트워크

```powershell
Get-NetIPAddress
Get-NetIPConfiguration
Get-NetRoute
Get-NetTCPConnection                    # ss / netstat
Test-NetConnection -Port 443 google.com

ipconfig /all
ipconfig /flushdns
netstat -ano
nslookup google.com
tracert google.com

# Firewall
Get-NetFirewallRule
New-NetFirewallRule -DisplayName "Block 8080" -Direction Inbound -LocalPort 8080 -Protocol TCP -Action Block
```

---

## 12. 가상화 — Hyper-V

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
Get-VM
Start-VM Linux
```

WSL2, Docker Desktop 의 백엔드.

---

## 13. Docker / Container

- Docker Desktop = WSL2 또는 Hyper-V 안 Linux + Windows container
- Windows container (Hyper-V isolation 또는 process isolation)
- 일반적으로 Linux container 가 표준

```powershell
docker ps
docker run hello-world
```

---

## 14. 보안 모델

- **SID** (Security Identifier) — UID / GID 비슷
- **ACL / DACL / SACL** — 권한
- **Token** — process 의 권한 set
- **UAC** (User Account Control) — sudo 비슷
- **Defender** — antivirus + EDR
- **BitLocker** — 디스크 암호화
- **Credential Guard** — VBS

---

## 15. 부팅

```
UEFI → bootloader (bootmgfw.efi) → winload.efi → ntoskrnl.exe → SMSS → CSRSS → WININIT/SERVICES → ...
```

```cmd
bcdedit                                 # boot configuration
```

---

## 16. 진단 도구

```cmd
perfmon                                  # Performance Monitor
resmon                                   # Resource Monitor
taskmgr                                  # Task Manager
msinfo32

# Sysinternals (Microsoft)
procexp                                  # Process Explorer
procmon                                  # Process Monitor
autoruns                                 # 시작 항목
psexec
handle
```

Linux 의 strace 동등 = procmon. ps 동등 = procexp.

---

## 17. Windows Server 의 추가

- AD (Active Directory) — LDAP + Kerberos
- DNS / DHCP / IIS / SMB
- Group Policy
- WSUS (update server)
- Failover Cluster
- Storage Spaces / Storage Replica

---

## 18. .NET / Win32 API

```c
HANDLE h = CreateFile(...);
WriteFile(h, ...);
CloseHandle(h);
```

```csharp
File.WriteAllText("file.txt", "hello");
```

Linux 응용 포팅 시 — Win32 API 또는 WSL.

---

## 19. WSL2 vs Linux 네이티브

| | WSL2 | 네이티브 Linux |
| --- | --- | --- |
| 커널 | 진짜 Linux (Hyper-V) | Linux |
| 파일 I/O (Linux fs) | 빠름 | 빠름 |
| Windows fs (`/mnt/c`) | 느림 | n/a |
| 통합 | VS Code Remote 등 강력 | n/a |
| GPU | DirectX/CUDA 지원 | native |
| Docker | Docker Desktop 통합 | native |

WSL2 = Windows 안의 Linux dev 표준.

---

## 20. 함정

### 20.1 권한 (UAC)
관리자 권한 prompt — script 자동화 어려움.

### 20.2 path 의 `\` vs `/`
Windows `\` — PowerShell / WSL 에선 `/` 도 OK.

### 20.3 line ending (CRLF vs LF)
git config `core.autocrlf`. .gitattributes.

### 20.4 Registry 변경 후 reboot
일부 변경 즉시 X.

### 20.5 antivirus 의 영향
실시간 검사 — file I/O / build 매우 느려짐. 제외 디렉토리 설정.

### 20.6 WSL `/mnt/c` 의 성능
cross-fs — 매우 느림. 프로젝트는 `~` (Linux fs).

### 20.7 Defender / Credential Guard + nested virt
충돌 → Docker / VirtualBox 안 됨.

### 20.8 windows binary 의 path length
260 자 한계 (long path 옵션으로 해결).

---

## 21. 학습 자료

- **Windows Internals** (Russinovich) — 가장 깊음
- **Sysinternals Suite**
- **PowerShell Documentation**
- **WSL Documentation**

---

## 22. 관련

- [[nt-architecture]]
- [[powershell]]
- [[wsl]]
- [[../operating-system|↑ OS hub]]
