---
title: "systemd — Service Manager"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T16:20:00+09:00
tags:
  - operating-system
  - linux
  - systemd
---

# systemd

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | unit / service / target / journal |

**[[linux|↑ Linux hub]]**

---

## 1. 한 줄

Lennart Poettering, 2010. Linux 의 **PID 1 + service manager + log + 부팅 + 자원 관리 + container + ...**.
대부분의 distro 의 기본 init.

---

## 2. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Unit** | 모든 관리 대상의 단위 |
| **Service** | 데몬 / 응용 (.service) |
| **Target** | unit 의 그룹 (= runlevel) |
| **Timer** | cron 대체 |
| **Socket** | socket activation |
| **Mount / Automount** | fstab 대체 |
| **Path** | file watch trigger |
| **Slice** | cgroup hierarchy |
| **Scope** | external process group |

---

## 3. 유닛 파일 위치

```
/lib/systemd/system/           — 패키지 제공
/etc/systemd/system/            — 관리자 override (우선)
~/.config/systemd/user/         — 사용자 단위

systemctl cat nginx              # 효과 file 보기
systemctl edit nginx             # override
```

---

## 4. 기본 명령

```bash
systemctl status nginx
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl reload nginx
systemctl enable nginx                # 부팅 시 자동
systemctl disable nginx
systemctl enable --now nginx           # enable + start

systemctl list-units --type=service
systemctl list-units --failed
systemctl list-unit-files

systemctl daemon-reload                # unit 변경 후
systemctl is-active nginx
systemctl is-enabled nginx
systemctl is-failed nginx
```

---

## 5. Service 파일

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My App
After=network.target postgresql.service
Requires=postgresql.service
Wants=redis.service

[Service]
Type=simple
ExecStart=/usr/bin/myapp
ExecReload=/bin/kill -HUP $MAINPID
WorkingDirectory=/srv/myapp
User=myapp
Group=myapp
EnvironmentFile=/etc/myapp/env
Restart=on-failure
RestartSec=5s
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

---

## 6. Type

| Type | 의미 |
| --- | --- |
| `simple` (기본) | ExecStart 가 main process. fg 실행 권장 |
| `exec` | simple + 환경 준비까지 기다림 |
| `forking` | fork 후 자식이 main (옛 데몬) |
| `oneshot` | 한 번 실행 후 끝 |
| `notify` | sd_notify("READY=1") 호출 |
| `dbus` | D-Bus name 잡으면 ready |
| `idle` | 다른 service 시작 후 |

→ 현대 응용은 **simple** 이 표준. fork 가 아닌 fg.

---

## 7. Restart

```ini
Restart=on-failure              # no / always / on-success / on-failure / on-abnormal / on-watchdog
RestartSec=5s
StartLimitIntervalSec=60
StartLimitBurst=5                # 60초에 5번 fail 시 멈춤
```

---

## 8. Dependency

```ini
After=...      # 순서 (시작 순서)
Before=...
Requires=...   # 강한 의존 (실패 시 같이 fail)
Wants=...      # 약한 (try, 실패해도 계속)
Conflicts=...
BindsTo=...    # Requires + stop also propagates
PartOf=...     # restart / stop propagates
```

---

## 9. Target — runlevel 대체

```bash
systemctl get-default
# graphical.target

systemctl list-units --type=target
# basic.target
# multi-user.target
# graphical.target
# network.target
# network-online.target
# sysinit.target

systemctl isolate rescue.target          # 그 target 으로 전환
```

| Target | 옛 runlevel |
| --- | --- |
| poweroff.target | 0 |
| rescue.target | 1 (single-user) |
| multi-user.target | 3 (text) |
| graphical.target | 5 (X) |
| reboot.target | 6 |

---

## 10. Timer (cron 대체)

```ini
# myapp-cleanup.timer
[Unit]
Description=Cleanup daily

[Timer]
OnCalendar=daily
Persistent=true
RandomizedDelaySec=600

[Install]
WantedBy=timers.target

# myapp-cleanup.service (oneshot)
[Service]
Type=oneshot
ExecStart=/usr/bin/myapp cleanup
```

```bash
systemctl list-timers
```

자세히 → [[cron-systemd-timer]]

---

## 11. Socket Activation

```ini
# myapp.socket
[Socket]
ListenStream=8080
Accept=false

[Install]
WantedBy=sockets.target

# myapp.service
[Service]
ExecStart=/usr/bin/myapp
StandardInput=socket
```

- systemd 가 socket 열고 응용 시작 시 FD 전달
- 응용 시작 전 socket ready → 의존성 단순
- 사용 시작 시만 daemon 실행 (xinetd 비슷)

---

## 12. 자원 제한 (cgroup 통합)

```ini
[Service]
MemoryMax=2G
MemoryHigh=1.5G
CPUQuota=200%               # 2 코어
CPUWeight=100
IOWeight=100
TasksMax=512
LimitNOFILE=65535
LimitNPROC=1024
```

자세히 → [[../virtualization/cgroups#6-systemd-와-cgroup]]

---

## 13. Sandbox 옵션

```ini
[Service]
DynamicUser=yes
PrivateTmp=yes
PrivateNetwork=yes
PrivateDevices=yes
PrivateUsers=yes
ProtectSystem=strict
ProtectHome=true
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectControlGroups=yes
ProtectClock=yes
ProtectHostname=yes
ProtectProc=invisible
ReadWritePaths=/var/log/myapp

NoNewPrivileges=yes
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE
SystemCallFilter=@system-service
```

자세히 → [[../security/sandbox#6-systemd-의-service-sandbox]]

---

## 14. journalctl — 로그

```bash
journalctl -u nginx -f                # follow
journalctl -u nginx --since "1 hour ago"
journalctl -u nginx --until "10 min ago"
journalctl --since today
journalctl -p err                      # priority err 이상
journalctl -k                          # kernel
journalctl -b                          # 이번 부팅
journalctl -b -1                        # 직전 부팅
journalctl -xe                         # 마지막 + explain
journalctl --vacuum-time=1month
```

자세히 → [[logging]]

---

## 15. User Service

```bash
# ~/.config/systemd/user/
systemctl --user start myapp.service
systemctl --user enable myapp.service
loginctl enable-linger username        # 로그아웃 후에도 실행
```

데스크탑 / 사용자 별 daemon.

---

## 16. Drop-in / Override

```bash
systemctl edit nginx
# /etc/systemd/system/nginx.service.d/override.conf
```

원본 unit 안 건드리고 일부만 변경.

---

## 17. Logind / Login Management

```bash
loginctl list-sessions
loginctl session-status
loginctl terminate-session 5
loginctl enable-linger user
```

- TTY / GUI 로그인
- inhibitor lock
- power button 동작

---

## 18. machinectl / nspawn

systemd 의 container.

```bash
sudo systemd-nspawn -D /var/lib/machines/mycontainer
machinectl list
```

자세히 → [[../virtualization/container]]

---

## 19. networkd / resolved

```bash
networkctl
networkctl list
resolvectl status
resolvectl query example.com
```

NetworkManager 대안 (서버 / cloud).

---

## 20. systemd-analyze

```bash
systemd-analyze
systemd-analyze blame
systemd-analyze critical-chain
systemd-analyze plot > boot.svg
systemd-analyze verify myapp.service
systemd-analyze security myapp.service     # sandbox 점수
```

→ 부팅 시간 분석 + unit 검증.

---

## 21. 함정

### 21.1 daemon-reload 누락
unit 수정 후 반영 X.

### 21.2 Type=forking + double-fork
PID 추적 실패. Type=simple 권장.

### 21.3 Environment 따옴표
```ini
Environment=PATH=/usr/bin
Environment="MULTI_WORD=hello world"
```

### 21.4 cron 의존
systemd timer 권장. 같이 쓰면 혼동.

### 21.5 enable 만 + restart 안 함
패키지 update 후 service 재시작 필요.

### 21.6 user service 의 linger
로그아웃 시 죽음.

### 21.7 ProtectSystem=strict 후 write
ReadWritePaths 추가 필요.

### 21.8 journalctl 의 volatile
`/var/log/journal` 디렉토리 없으면 RAM 만 (재부팅 시 사라짐).

---

## 22. 학습 자료

- **systemd.unit / .service / .timer (5)** man pages
- **systemd.exec(5)** — sandbox
- **Lennart Poettering blog** — 0pointer.net
- **digitalocean / arch wiki** systemd

---

## 23. 관련

- [[boot-process]]
- [[cron-systemd-timer]]
- [[logging]]
- [[../virtualization/cgroups]]
- [[../security/sandbox]]
- [[linux]] — Linux hub
