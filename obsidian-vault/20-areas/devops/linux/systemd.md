---
title: "systemd — service / unit / journalctl"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T06:45:00+09:00
tags: [devops, linux, systemd]
---

# systemd — service / unit / journalctl

**[[linux|↑ linux]]**

---

## 1. systemd 란

- 현대 Linux 의 init 시스템 (PID 1).
- service 관리 + log + boot + cgroup + timer + socket activation.
- 거의 모든 distro (Ubuntu 16+ / RHEL 7+ / Debian 8+).

→ 비교: SysV init (`/etc/init.d/`) / upstart / runit / OpenRC.

---

## 2. systemctl 기본

```bash
systemctl status nginx
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl reload nginx        # SIGHUP (config reload)

systemctl enable nginx        # boot 시 자동 시작
systemctl disable nginx
systemctl enable --now nginx  # enable + start

systemctl is-active nginx
systemctl is-enabled nginx
systemctl is-failed nginx

systemctl list-units --type=service
systemctl list-units --failed
systemctl list-unit-files

systemctl daemon-reload       # unit file 변경 후
```

---

## 3. unit file 작성

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My App
After=network.target postgresql.service
Requires=postgresql.service

[Service]
Type=simple                   # 또는 forking, oneshot, notify
User=app
Group=app
WorkingDirectory=/opt/myapp
Environment="JAVA_OPTS=-Xmx512m"
EnvironmentFile=/etc/myapp/env
ExecStart=/usr/bin/java -jar /opt/myapp/app.jar
ExecReload=/bin/kill -SIGHUP $MAINPID
ExecStop=/bin/kill -SIGTERM $MAINPID
Restart=on-failure
RestartSec=5s
TimeoutStopSec=30s
LimitNOFILE=65535
StandardOutput=journal
StandardError=journal

# 보안 강화
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
ReadWritePaths=/var/log/myapp

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now myapp
```

---

## 4. Service Type

| Type | 무엇 | 언제 |
| --- | --- | --- |
| **simple** | ExecStart 자체가 main process | 일반 (default) |
| **forking** | ExecStart 가 daemon 으로 fork | 전통 daemon (nginx 등) |
| **oneshot** | 한 번 실행 후 종료 | script |
| **notify** | sd_notify() 호출로 ready 알림 | systemd-aware |
| **idle** | 다른 unit 후 실행 | boot 메시지 정리 |

---

## 5. dependency

```ini
After=network.target          # network 후 시작 (순서)
Wants=postgresql.service      # 약한 의존 (있으면 시작)
Requires=postgresql.service   # 강한 의존 (실패 시 본인도 실패)
BindsTo=device.unit           # device 사라지면 stop
Conflicts=other.service       # 같이 못 실행
PartOf=parent.target          # 부모 restart 시 같이
```

→ **Requires + After** 가 가장 흔한 조합.

---

## 6. journalctl (log)

```bash
journalctl -u myapp                   # myapp service 의 log
journalctl -u myapp -f                # follow (tail -f)
journalctl -u myapp --since "1 hour ago"
journalctl -u myapp --since "2026-05-15 10:00" --until "11:00"
journalctl -u myapp -p err            # err 이상만 (emerg/alert/crit/err/warn/notice/info/debug)
journalctl -u myapp --no-pager        # less 안 쓰고
journalctl -u myapp -o json           # JSON 출력
journalctl -k                          # kernel
journalctl -b                          # 현재 boot
journalctl --disk-usage
```

---

## 7. journal 영구 저장

```bash
# /etc/systemd/journald.conf
[Journal]
Storage=persistent
SystemMaxUse=1G
SystemKeepFree=500M
MaxRetentionSec=7day
```

```bash
sudo systemctl restart systemd-journald
```

---

## 8. timer (cron 대체)

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Backup

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
```

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Daily backup

[Timer]
OnCalendar=daily              # 매일 0시
# 또는 OnCalendar=*-*-* 02:00:00    # 매일 2시
# 또는 OnCalendar=Mon..Fri 09:00
Persistent=true                # 부팅 중 missed 면 catch up
RandomizedDelaySec=5min

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl enable --now backup.timer
systemctl list-timers
```

→ cron 보다 log / dependency / monitoring 우수.

---

## 9. socket activation

```ini
# myapp.socket
[Socket]
ListenStream=8080
Accept=false

[Install]
WantedBy=sockets.target
```

```ini
# myapp.service (socket 으로 시작 시)
[Service]
Type=simple
ExecStart=/usr/bin/myapp
```

→ traffic 올 때만 service 시작 (lazy).

---

## 10. resource 제한

```ini
[Service]
CPUQuota=50%
MemoryMax=512M
TasksMax=200
LimitNOFILE=65535
IOWeight=200
```

→ systemd 가 cgroup 으로 enforcement.

---

## 11. drop-in (override)

```bash
sudo systemctl edit nginx
# → /etc/systemd/system/nginx.service.d/override.conf 생성
```

```ini
[Service]
LimitNOFILE=100000
```

→ 원본 unit 건드리지 않고 일부만 override.

---

## 12. 함정

1. **daemon-reload 안 함** — unit 변경 후 적용 안 됨.
2. **Type=forking + main PID 추적 X** — 잘못된 PID 추적.
3. **Restart=always** + crash loop — 계속 재시작.
4. **logging journal 없음** — `StandardOutput=null` 로 log 사라짐.
5. **dependency 순환** — boot 행.
6. **enable 했는데 start 안 함** — `enable --now` 또는 `start` 별도.

---

## 13. 관련

- [[linux|↑ linux]]
- [[process-management]]
- [[../docker/docker|↗ docker]]
