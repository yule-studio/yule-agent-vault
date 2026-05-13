---
title: "Cron / systemd Timer — 예약 작업"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T16:55:00+09:00
tags:
  - operating-system
  - linux
  - cron
  - systemd
---

# Cron / systemd Timer

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | cron / timer 비교 |

**[[linux|↑ Linux hub]]**

---

## 1. 두 방법

| | cron | systemd timer |
| --- | --- | --- |
| 도구 | crond / cronie | systemd |
| 문법 | 5 필드 | OnCalendar / OnUnitActive |
| 로그 | mail / syslog | journal |
| 정확도 | 분 단위 | μs 가능 |
| 재시도 / 의존성 | X | ✅ |
| 누락 catchup | X | Persistent=true |
| 사용자 단위 | crontab -u | --user |

→ 현대 = **systemd timer** 권장. cron 은 친숙해서 여전히 흔함.

---

## 2. crontab

```bash
crontab -e                       # 자기 cron 편집
crontab -l                       # 보기
crontab -r                       # 모두 제거

sudo crontab -e -u alice         # 다른 사용자
```

### 2.1 문법

```
m h dom mon dow  command
* * *   *   *

m   minute (0-59)
h   hour   (0-23)
dom day of month (1-31)
mon month  (1-12)
dow day of week (0-7, 0/7=Sun)
```

```cron
# 매분
* * * * * /usr/bin/script.sh

# 매시간 0분
0 * * * * cmd

# 매일 새벽 3시 30분
30 3 * * * /usr/bin/backup.sh

# 평일 9-18시 매 15분
*/15 9-18 * * 1-5 cmd

# 매월 1일 자정
0 0 1 * * cmd

# 시간 외 짧은 표기
@reboot     /opt/myapp/start.sh
@hourly     cmd
@daily      cmd
@weekly     cmd
@monthly    cmd
@yearly     cmd
```

---

## 3. /etc/cron.*

```
/etc/cron.d/         — 패키지 / 응용
/etc/cron.daily/     — 매일 (anacron)
/etc/cron.hourly/
/etc/cron.weekly/
/etc/cron.monthly/

/var/spool/cron/<user>/    — 사용자 crontab
```

`/etc/crontab` (시스템) 은 user 컬럼 포함:

```
30 3 * * * root /usr/bin/script.sh
```

---

## 4. cron 환경

```cron
SHELL=/bin/bash
PATH=/usr/bin:/bin:/usr/sbin
MAILTO=admin@example.com
HOME=/root

30 3 * * * /usr/bin/script.sh
```

⚠️ default PATH 작음 — 명령은 **절대 경로** 또는 PATH 명시.
환경 변수 로그인 시와 다름 — script 가 다르게 동작.

---

## 5. cron 로그

```bash
# Debian/Ubuntu
journalctl -u cron
grep CRON /var/log/syslog

# RHEL
journalctl -u crond

# 응용 로그 capture
30 3 * * * /usr/bin/script.sh >> /var/log/script.log 2>&1
```

기본은 mail (root 의 local mail) 로 stdout 전송. 별로 안 봄.

---

## 6. anacron

`cron.daily` 등은 anacron 처리. 시스템이 꺼져 있다 켜져도 missed job 실행.

```
/etc/anacrontab:
period   delay   job-identifier   command
1        5       cron.daily       run-parts /etc/cron.daily
7        25      cron.weekly      run-parts /etc/cron.weekly
@monthly 45      cron.monthly     run-parts /etc/cron.monthly
```

데스크탑 / 노트북 친화.

---

## 7. systemd Timer

### 7.1 service unit

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Daily backup

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
User=backup
StandardOutput=journal
StandardError=journal
```

### 7.2 timer unit

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Run backup daily

[Timer]
OnCalendar=daily
Persistent=true              # 놓친 실행 catchup
RandomizedDelaySec=600        # 0-10분 랜덤 지연

[Install]
WantedBy=timers.target
```

### 7.3 활성

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now backup.timer
systemctl list-timers
systemctl status backup.timer
```

---

## 8. OnCalendar 문법

```
OnCalendar=Mon..Fri 09:00       # 평일 9시
OnCalendar=*-*-* 03:30:00        # 매일 3:30
OnCalendar=hourly
OnCalendar=daily
OnCalendar=weekly
OnCalendar=monthly
OnCalendar=*-*-1 00:00:00        # 매월 1일
OnCalendar=*:0/5                  # 매 5분
OnCalendar=Mon *-*-* 03:00        # 매주 월 3시
OnCalendar=2026-12-31 23:59:59    # 한 번
```

검증:
```bash
systemd-analyze calendar 'Mon..Fri 09:00'
# Next elapse: Mon 2026-05-19 09:00:00
```

---

## 9. 다른 trigger

```ini
[Timer]
OnBootSec=15min                   # 부팅 후 15분
OnUnitActiveSec=1h                 # 마지막 활성 후 1시간
OnUnitInactiveSec=1h
OnStartupSec=1min
OnUnitActiveSec=30s
AccuracySec=1s                      # 기본 1분
```

`OnBootSec` + `OnUnitActiveSec` = cron 의 `@hourly` 비슷.

---

## 10. user timer

```bash
# ~/.config/systemd/user/myapp.timer
systemctl --user enable --now myapp.timer
loginctl enable-linger $USER       # 로그아웃 후 유지
```

---

## 11. cron vs systemd timer 비교

### 11.1 cron 장점
- 간단 / 익숙
- 모든 Linux 표준
- 한 줄 표기

### 11.2 systemd timer 장점
- service 와 의존성 / 환경 / sandbox / cgroup
- journal log
- catchup (`Persistent=true`)
- 정확한 시간
- monitoring (`systemctl status` / `list-timers`)
- network online 후 실행 가능

---

## 12. 자주 보는 함정

### 12.1 cron 의 PATH
default PATH 가 작음. 명령은 절대 경로.

### 12.2 % 문자
cron 에서 `%` = newline. escape `\%` 필요.
```cron
0 3 * * * date +\%F
```

### 12.3 환경 변수
login shell 과 다름. cron 안에서 source ~/.bashrc 등.

### 12.4 timezone
cron 은 system timezone. systemd timer = `Timezone=Asia/Seoul`.

### 12.5 출력
cron 의 mail → root 만 봄. 명시적 `>> log 2>&1`.

### 12.6 동시 실행 방지
같은 job 두 번 = 충돌. `flock` 또는 `systemd-cat` + dependency.

```bash
* * * * * flock -n /tmp/script.lock /usr/bin/script.sh
```

### 12.7 Daylight Saving
시계 변경 시 작업 누락 / 두 번. systemd timer 더 안전.

### 12.8 catchup 안 됨
cron — system off 동안 누락. anacron 또는 systemd Persistent.

### 12.9 systemd timer 변경 후 daemon-reload
누락 시 옛 schedule.

---

## 13. 실전 예

### 13.1 매시간 ETL

```ini
# etl.service
[Service]
Type=oneshot
ExecStart=/srv/etl/run.sh
WorkingDirectory=/srv/etl
User=etl

# etl.timer
[Timer]
OnCalendar=hourly
RandomizedDelaySec=120

[Install]
WantedBy=timers.target
```

### 13.2 매일 새벽 백업

```ini
# backup.timer
[Timer]
OnCalendar=03:30
RandomizedDelaySec=600
Persistent=true
```

### 13.3 cron 으로 같은 것

```cron
30 3 * * * /srv/backup/run.sh >> /var/log/backup.log 2>&1
```

---

## 14. 작업 큐 (cron 대안)

거대 / 분산 작업 = job queue:

- Celery (Python)
- Sidekiq (Ruby)
- BullMQ (Node)
- Quartz (Java)
- Kubernetes CronJob

K8s:
```yaml
apiVersion: batch/v1
kind: CronJob
metadata: { name: backup }
spec:
  schedule: "30 3 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: backup:latest
```

---

## 15. 학습 자료

- `man 5 crontab` / `man 8 cron`
- `man 7 systemd.timer` / `systemd-analyze calendar`
- **Pro Linux System Administration**

---

## 16. 관련

- [[systemd]]
- [[logging]]
- [[linux]] — Linux hub
