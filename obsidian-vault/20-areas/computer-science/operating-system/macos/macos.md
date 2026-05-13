---
title: "macOS (Hub)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T17:30:00+09:00
tags:
  - operating-system
  - macos
  - hub
---

# macOS (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | macOS 의 OS 측면 |

**[[../operating-system|↑ OS hub]]**

---

## 1. 한 줄

Apple 의 데스크탑 OS. XNU 커널 (Mach + BSD + IOKit) + Darwin userland + Cocoa GUI.

2001 macOS X 10.0 → 현재 26.x.

---

## 2. XNU 커널

```
Mach (microkernel)
  + BSD layer (UNIX 인터페이스)
  + IOKit (C++ driver framework)
  + Kext (kernel extension, 점차 deprecated)
```

→ "hybrid" — microkernel 의 IPC + monolithic 의 BSD 통합.

```bash
uname -a
sysctl -a | grep kern.version
```

---

## 3. Darwin

오픈소스 부분. macOS / iOS / iPadOS / tvOS / watchOS 의 공통 토대.

- BSD-derived userland (`ls`, `cp`, ... — BSD 변형, GNU 와 다름)
- launchd (PID 1)
- Mach IPC
- dyld

```bash
which ls
ls --version    # ❌ BSD 라 다름
# 옛 BSD 옵션이 GNU 와 다름

# GNU 도구 설치
brew install coreutils    # ls = gls
```

---

## 4. launchd

PID 1 — systemd 비슷.

```bash
launchctl list
launchctl load ~/Library/LaunchAgents/com.example.plist
launchctl unload ...
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/x.plist

# 위치
/Library/LaunchDaemons/      — 시스템 root
/Library/LaunchAgents/        — 시스템 user (login 후)
~/Library/LaunchAgents/       — user
/System/Library/...           — Apple
```

### 4.1 plist

```xml
<?xml version="1.0" encoding="UTF-8"?>
<plist version="1.0">
<dict>
    <key>Label</key>           <string>com.example.app</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/myapp</string>
    </array>
    <key>RunAtLoad</key>        <true/>
    <key>KeepAlive</key>        <true/>
    <key>StartCalendarInterval</key>
    <dict><key>Hour</key><integer>3</integer></dict>
</dict>
</plist>
```

systemd 의 `Type=simple Restart=always` + cron 동시.

---

## 5. APFS

자세히 → [[../filesystem/ext-xfs-zfs#7-apfs-macos]]

```bash
diskutil apfs list
diskutil list
diskutil info /
tmutil snapshot
tmutil listlocalsnapshots /
```

---

## 6. SIP (System Integrity Protection)

```bash
csrutil status
```

root 라도 `/System`, `/usr` 수정 X.

```bash
# Recovery mode 에서만 (Cmd+R)
csrutil disable
```

⚠️ 일반적으로 켜둠.

---

## 7. 보안 모델

- **Gatekeeper** — 서명되지 않은 binary 차단
- **Notarization** — Apple 공증
- **Sandbox** — App Store 응용 격리
- **TCC (Transparency Consent & Control)** — 카메라 / 마이크 / 디스크 권한
- **SIP** — system 보호
- **APFS encryption** — FileVault

```bash
# Gatekeeper bypass (임시)
sudo spctl --master-disable
```

---

## 8. dyld — Dynamic Linker

```bash
otool -L /usr/bin/ls           # 종속 라이브러리
# /usr/lib/libSystem.B.dylib

DYLD_PRINT_LIBRARIES=1 ./app
DYLD_LIBRARY_PATH=...           # SIP 때문에 의도와 달리 동작
```

`*.dylib` (Linux 의 `.so`), `*.dyld` (실행 파일 framework).

---

## 9. Process / IPC

- **Mach port** — IPC 핵심
- BSD signal / pipe / socket (UNIX)
- XPC (high-level RPC, sandbox 친화)
- launchd 가 service 등록

```bash
ps aux
top -o cpu
fs_usage
dtruss / dtrace        # strace 동등
sample $PID 10          # 10s sampling
spindump $PID           # hang 분석
```

---

## 10. Homebrew

표준 패키지 매니저.

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

brew install nginx
brew services start nginx
brew services list

brew install --cask firefox     # GUI 응용
brew list / brew search / brew info nginx
brew upgrade
brew cleanup
brew doctor
```

`/opt/homebrew` (Apple Silicon) / `/usr/local` (Intel).

---

## 11. Activity Monitor / 도구

```bash
top -o cpu
top -o mem
vm_stat 1                 # vmstat 동등
iostat 1
fs_usage -w cat            # 파일 syscall trace
sample $PID 10             # CPU sampling
spindump                    # 시스템 hang
sysdiagnose                 # 전체 진단 (큰 archive)
```

---

## 12. 네트워크

```bash
ifconfig                  # 옛, 여전히 일반적
ipconfig getifaddr en0    # 인터페이스 IP
networksetup -listallnetworkservices

scutil --dns               # DNS 정보
scutil --proxy
route -n get default

# 방화벽
sudo pfctl -d              # disable
sudo pfctl -e              # enable
sudo pfctl -sr             # rules
```

`pf` 패킷 필터 + Application Firewall (System Preferences).

---

## 13. UNIX 호환 + 차이

| | macOS (BSD) | GNU/Linux |
| --- | --- | --- |
| `ls -l` | 같음 | 같음 |
| `ls -G` | color (BSD) | `ls --color` |
| `sed -i ''` | 백업 필수 | `sed -i` |
| `xargs -P` | 같음 | 같음 |
| `cp -a` | BSD 없음 | GNU |
| `date` | BSD | GNU 더 풍부 |
| `readlink -f` | 다름 | GNU |
| `find -print0` / `xargs -0` | 같음 | 같음 |

→ 작은 차이 — `coreutils` / `findutils` 설치 시 GNU 도구.

---

## 14. Docker on Mac

호스트 macOS 가 Linux container 직접 실행 X — VM 안에서:
- Docker Desktop — Hyperkit / Apple Virtualization framework
- Colima / Lima — 오픈소스 대안
- Podman Machine

```bash
brew install colima docker docker-compose
colima start
docker ps
```

→ Mac 의 file system 마운트 = slow (cross-VM I/O).

---

## 15. Apple Silicon (M1/M2/M3/M4)

```bash
uname -m
# arm64 (Apple Silicon)
# x86_64 (Intel)
```

- arm64 native
- Rosetta 2 — x86_64 binary emulation (자동)
- universal binary (`fat` arch)

```bash
file /usr/bin/ls
# Mach-O 64-bit executable arm64 + x86_64

arch -arm64 ./app
arch -x86_64 ./app
```

---

## 16. iOS / iPadOS 와의 관계

같은 XNU + Darwin + 다른 GUI / sandbox 모델.
mac 의 SwiftUI + macOS-only API.

---

## 17. 자주 보는 차이

- 권한 prompt — TCC (디스크 / 카메라 / 마이크)
- root user 기본 비활성 — `sudo` 사용
- /etc/hosts 는 같지만 일부 caching
- `~/.zshrc` 가 기본 shell (Catalina 이상)

---

## 18. 함정

### 18.1 BSD 도구의 옵션 차이
script 가 Linux 가정 — `gsed`, `gdate`, `gxargs` 또는 sh 호환.

### 18.2 Docker / VM 의 디스크 I/O
host fs mount = 느림. volume 사용.

### 18.3 SIP + system 파일
root 도 X. 대안 — sudo not enough.

### 18.4 dyld 환경변수
SIP 때문에 sub-process 에 전파 X.

### 18.5 codesign / Notarization
배포 시 필수 — 안 하면 사용자에게 경고.

### 18.6 Recovery mode 접근
M1+ = Power button 길게. Intel = Cmd+R 부팅.

### 18.7 Time Machine + APFS local snapshot
디스크 풀 보임. 자동 정리되지만 시간 걸림.

---

## 19. 학습 자료

- **macOS Internals** — Jonathan Levin (가장 깊음)
- **Apple Developer docs**
- `man launchctl` / `man launchd.plist`
- **Real World macOS Internals** blog 글

---

## 20. 관련

- [[../filesystem/ext-xfs-zfs]] — APFS
- [[../operating-system|↑ OS hub]]
