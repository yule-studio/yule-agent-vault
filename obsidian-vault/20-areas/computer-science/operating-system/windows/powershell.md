---
title: "PowerShell — Object-based Shell"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T17:45:00+09:00
tags:
  - operating-system
  - windows
  - powershell
---

# PowerShell

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | cmdlet + pipeline + scripting |

**[[windows|↑ Windows hub]]**

---

## 1. 한 줄

Microsoft 의 shell + 스크립트 언어. **object 기반 pipeline** — bash 의 text 와 다름.
.NET 통합 + cross-platform (PowerShell Core 7+).

---

## 2. 두 버전

| | Windows PowerShell 5.1 | PowerShell Core 7+ |
| --- | --- | --- |
| 기반 | .NET Framework | .NET Core |
| platform | Windows | Win / Linux / macOS |
| 미래 | 유지보수 | 적극 개발 |

`pwsh` 가 새 binary, `powershell` 이 옛 (Windows).

---

## 3. Cmdlet — `Verb-Noun`

```
Get-Process
Stop-Service
Set-Location
New-Item
Remove-Item
Test-Connection
```

`Get` / `Set` / `Stop` / `Start` / `Remove` / `New` / `Test` / `Invoke` 등.

```powershell
Get-Command -Verb Get | Select-Object -First 10
Get-Command -Noun Process
```

---

## 4. Object Pipeline

bash:
```bash
ps aux | grep nginx | awk '{print $2}' | xargs kill
```

PowerShell:
```powershell
Get-Process nginx | Stop-Process
```

object 직접 전달 — 파싱 / 가공 불필요.

```powershell
Get-Process | Where-Object { $_.CPU -gt 100 } | Sort-Object CPU -Descending | Select-Object -First 5

# 단축 (alias)
ps | ? { $_.CPU -gt 100 } | sort CPU -Desc | select -First 5
```

---

## 5. 자주 쓰는 cmdlet

### 5.1 File / Path

```powershell
Get-ChildItem                            # ls
Set-Location                              # cd
Get-Content file.txt                      # cat
Set-Content -Path file.txt -Value "hi"
Add-Content -Path file.txt -Value "more"
Copy-Item / Move-Item / Remove-Item
New-Item -ItemType Directory -Path mydir
Test-Path path
Resolve-Path .                            # realpath
```

### 5.2 Process / Service

```powershell
Get-Process
Get-Process nginx
Stop-Process -Name nginx -Force
Start-Process notepad

Get-Service
Get-Service nginx
Start-Service / Stop-Service / Restart-Service
```

### 5.3 Network

```powershell
Test-NetConnection google.com -Port 443
Resolve-DnsName google.com
Get-NetIPAddress
Get-NetTCPConnection
Invoke-WebRequest https://...
Invoke-RestMethod https://api/url -Method POST -Body @{ key='value' } -ContentType 'application/json'
```

### 5.4 System

```powershell
Get-ComputerInfo
Get-Date
Get-EventLog -LogName Application -Newest 50
Get-WinEvent -LogName System
Get-Counter '\Memory\Available MBytes'
```

### 5.5 Registry

```powershell
Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion'
Set-ItemProperty -Path '...' -Name 'X' -Value 1
```

---

## 6. 변수 / 타입

```powershell
$x = 42                                  # int
$name = "alice"
$list = @(1, 2, 3)
$hash = @{ name = "alice"; age = 30 }
$obj = [PSCustomObject]@{ Name = 'A'; Score = 100 }

$x.GetType()                             # System.Int32
[int]"42"                                  # cast
```

---

## 7. 제어

```powershell
if ($x -eq 1) { ... } elseif ($x -eq 2) { ... } else { ... }

foreach ($f in Get-ChildItem) {
    Write-Host $f.Name
}

for ($i=0; $i -lt 10; $i++) { ... }

while (...) { ... }
do { ... } while (...)

switch ($x) {
    1 { 'one' }
    'a' { 'letter' }
    default { 'other' }
}
```

### 7.1 비교 연산자

```powershell
-eq / -ne / -lt / -gt / -le / -ge
-like / -notlike                          # wildcard
-match / -notmatch                         # regex
-contains / -in                            # list
-is / -isnot                               # type
```

---

## 8. Function / Script

```powershell
function Get-Hello {
    param(
        [Parameter(Mandatory)][string]$Name,
        [int]$Count = 1
    )
    1..$Count | ForEach-Object { "Hello $Name $_" }
}

Get-Hello -Name "alice" -Count 3
```

### 8.1 Script

```powershell
# script.ps1
param($Name)
Write-Host "Hi $Name"

# 실행 정책
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
.\script.ps1 -Name alice
```

---

## 9. Error Handling

```powershell
try {
    Get-Item nonexistent
} catch [System.IO.FileNotFoundException] {
    Write-Error $_.Exception.Message
} catch {
    Write-Error "Other: $_"
} finally {
    Write-Host "cleanup"
}

# 옵션
$ErrorActionPreference = 'Stop'           # 에러 시 throw
Get-Item x -ErrorAction SilentlyContinue
```

---

## 10. Pipeline 의 의미

```powershell
1..10 | ForEach-Object { $_ * 2 }
# 2 4 6 8 10 12 14 16 18 20

Get-Process | ForEach-Object -Begin { $i=0 } -Process { $i++; $_ } -End { Write-Host "Total: $i" }

# Group / Measure
Get-Process | Group-Object -Property ProcessName | Sort-Object Count -Descending | Select-Object -First 5

Get-ChildItem -Recurse -File | Measure-Object -Property Length -Sum
```

---

## 11. Format / Output

```powershell
Get-Process | Format-Table -Property Name, CPU, WS -AutoSize
Get-Process | Format-List Name, Id, Path

# JSON / CSV / XML
$x | ConvertTo-Json
$x | ConvertTo-Csv -NoTypeInformation
Import-Csv file.csv
ConvertFrom-Json $json

# 파일
Get-Process | Export-Csv -Path out.csv -NoTypeInformation
Get-Process | Out-File -FilePath out.txt
```

---

## 12. 자주 쓰는 단축

```powershell
ls          # Get-ChildItem
cd          # Set-Location
pwd         # Get-Location
cat / type  # Get-Content
echo        # Write-Output
where       # Where-Object
select      # Select-Object
sort        # Sort-Object
group       # Group-Object
measure     # Measure-Object
%           # ForEach-Object
?           # Where-Object
gci         # Get-ChildItem
```

---

## 13. Remoting (WinRM)

```powershell
# 활성
Enable-PSRemoting -Force

# 실행
Invoke-Command -ComputerName server01 -ScriptBlock { Get-Process }

# Session
$s = New-PSSession -ComputerName server01
Invoke-Command -Session $s -ScriptBlock { ... }
Remove-PSSession $s

# Linux 도 SSH 위에서
ssh user@host pwsh -c 'Get-Process'
```

---

## 14. Module / 패키지

```powershell
Get-Module -ListAvailable
Import-Module ActiveDirectory
Install-Module Az -Scope CurrentUser

Find-Module -Name PSReadLine
Update-Module
```

`PSGallery` 가 표준 repo.

---

## 15. PowerShell vs Bash

| | PowerShell | Bash |
| --- | --- | --- |
| 데이터 | object | text |
| 호환 | Win + cross | UNIX |
| 학습 | 명시적 verb-noun | 짧은 옛 명령 |
| 함수 | named param | positional |
| .NET 통합 | ✅ | ❌ |
| stream 처리 | object | line-by-line |

→ object pipeline 의 표현력 강함. text 처리는 bash 가 더 친숙.

---

## 16. Profile

```powershell
$PROFILE                                  # 보통 위치
notepad $PROFILE

# 자주 쓰는 alias / function 정의
```

`~/.bashrc` 동등.

---

## 17. 함정

### 17.1 ExecutionPolicy
기본 `Restricted` — 스크립트 실행 X. `RemoteSigned` 권장.

### 17.2 case-insensitive
대부분 명령 / 변수 case 무관. Linux 와 다름.

### 17.3 `>` 의 의미
text 가 아니라 default formatter — file 에 가독성 출력.
binary / 원본 = `Set-Content -Encoding Byte` 또는 `Out-File`.

### 17.4 외부 명령 + pipeline
`ls | grep` 처럼 bash 명령 mix 가능하지만 — bash output 은 text → object pipeline 깨짐. PowerShell native 사용.

### 17.5 `$_` 의 의미
ForEach / Where 안의 현재 element.

### 17.6 string interpolation
single quote `'$x'` 는 literal. double quote `"$x"` 가 expand.

### 17.7 변수 scope
`local`, `script`, `global` — bash 와 비슷하지만 명시적.

### 17.8 PSReadLine
탭 보완 / history. 최신 버전 권장.

---

## 18. 학습 자료

- **PowerShell Documentation** — learn.microsoft.com/powershell
- **PowerShell in Action** — Bruce Payette
- **PowerShell Gallery**

---

## 19. 관련

- [[windows]] — Windows hub
- [[wsl]]
- [[../linux/shell-commands]] — 비교
