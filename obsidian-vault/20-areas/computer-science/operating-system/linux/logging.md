---
title: "Linux Logging — journald / syslog / rsyslog"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T16:50:00+09:00
tags:
  - operating-system
  - linux
  - logging
---

# Linux Logging

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | journald / syslog / 운영 |

**[[linux|↑ Linux hub]]**

---

## 1. 두 종류

```
- syslog (전통) — 텍스트 파일 /var/log/
- journald (systemd) — 바이너리 binary index + query
```

→ 현대 distro = journald + (필요 시) rsyslog 통합.

---

## 2. journald

systemd-journald 가 모든 stdout/stderr 수집:

```bash
journalctl                            # 모든 로그
journalctl -u nginx                    # service 별
journalctl -u nginx -f                  # follow
journalctl -u nginx --since today
journalctl -u nginx --since "1 hour ago" --until "10 min ago"

journalctl _PID=1234
journalctl _UID=1000
journalctl _COMM=nginx
journalctl _SYSTEMD_UNIT=nginx.service

journalctl -p err                       # priority err 이상
journalctl -p 0..3                       # emerg ~ err
journalctl -b                            # 이번 부팅
journalctl -b -1                          # 직전 부팅
journalctl --list-boots

journalctl -k                            # kernel
journalctl -x                            # 설명 포함
journalctl -o json                       # JSON
journalctl -o json-pretty

journalctl --disk-usage
journalctl --vacuum-time=1month
journalctl --vacuum-size=2G
```

### 2.1 priority

```
0  emerg
1  alert
2  crit
3  err
4  warning
5  notice
6  info
7  debug
```

### 2.2 영구 / 휘발

```bash
# /etc/systemd/journald.conf
Storage=persistent        # /var/log/journal 에 영구
# auto = directory 존재하면 persistent, 없으면 volatile
# volatile = /run/log/journal (RAM, reboot 시 사라짐)
SystemMaxUse=2G
SystemMaxFileSize=100M
MaxRetentionSec=1month
```

```bash
sudo mkdir -p /var/log/journal
sudo systemctl restart systemd-journald
```

---

## 3. syslog (옛 / rsyslog)

```bash
# /var/log/syslog (Debian) / /var/log/messages (RHEL)
# /var/log/auth.log — 인증 / sudo / ssh
# /var/log/kern.log — kernel
# /var/log/dmesg
# /var/log/boot.log
# /var/log/cron / /var/log/maillog / ...
```

```bash
tail -f /var/log/syslog
grep -i error /var/log/auth.log
```

### 3.1 rsyslog

```
/etc/rsyslog.conf
/etc/rsyslog.d/

facility.priority   action
*.info;mail.none;authpriv.none;cron.none    /var/log/messages
authpriv.*                                   /var/log/secure
mail.*                                        -/var/log/maillog
*.emerg                                       :omusrmsg:*
```

### 3.2 syslog facility

```
kern, user, mail, daemon, auth, syslog, lpr,
news, uucp, cron, authpriv, ftp, local0-7
```

### 3.3 priority

```
emerg < alert < crit < err < warning < notice < info < debug
```

---

## 4. journald → syslog 통합

rsyslog 가 journald 에서 읽기:

```
module(load="imjournal")
input(type="imjournal" StateFile="imjournal.state")
```

→ journald 의 풍부함 + rsyslog 의 forwarding.

---

## 5. logrotate

```bash
# /etc/logrotate.d/myapp
/var/log/myapp/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0640 root adm
    postrotate
        systemctl reload myapp || true
    endscript
}
```

```bash
sudo logrotate -d /etc/logrotate.d/myapp        # dry run
sudo logrotate -f /etc/logrotate.d/myapp         # force
```

journald 는 자체 retention → logrotate 불필요.

---

## 6. 원격 / 중앙 집중

### 6.1 syslog forward

```
# /etc/rsyslog.d/forward.conf
*.* @@logserver:514          # TCP (RFC5424)
*.* @logserver:514            # UDP
```

### 6.2 journal remote

```bash
# /etc/systemd/journal-upload.conf
URL=https://log-server:19532
```

### 6.3 도구 / 플랫폼
- **Elastic Stack (ELK)** — Logstash / Filebeat / Elasticsearch
- **Fluentd / Fluent Bit** — CNCF, lightweight
- **Vector** — Rust, 빠름
- **Loki** + **Promtail** (Grafana)
- **Splunk**
- **Datadog Logs / Sumo Logic**

자세히 → [[../../database/elasticsearch/elasticsearch]]

---

## 7. 응용 로그 → stdout

12-factor:
> "Treat logs as event streams"

응용은 file 에 안 쓰고 **stdout / stderr** 로:
- systemd 가 journald 로 캡처
- container 가 stdout 캡처 → 외부로 forward
- 응용은 rotation 신경 X

→ K8s / cloud 의 표준.

---

## 8. structured log (JSON)

```json
{"level":"info","ts":"2026-05-14T16:50:00Z","msg":"req","path":"/","status":200,"ms":42}
```

- machine-readable
- elastic / loki 친화
- grep / jq 로 분석

```bash
journalctl -o json | jq 'select(.PRIORITY=="3")'
```

응용:
- **logrus / zap** (Go)
- **slog** (Go 1.21+)
- **structlog** (Python)
- **logback / log4j2** (Java) — JSON layout
- **bunyan / pino** (Node)

---

## 9. dmesg — kernel ring buffer

```bash
dmesg
dmesg -T                                  # human time
dmesg -w                                  # follow
dmesg -l err,warn                          # level
dmesg --since "1 hour ago"
dmesg -k                                   # kernel
```

→ kernel 의 모든 log. OOM / 디스크 / driver 디버그.

`/dev/kmsg` 가 backing.

---

## 10. audit

```bash
sudo auditctl -w /etc/passwd -p wa -k passwd-change
sudo ausearch -k passwd-change
sudo aureport

# /var/log/audit/audit.log
```

규정 준수 / 보안 감사.

---

## 11. 로그 분석 도구

```bash
grep / awk / sed / sort / uniq -c
jq                                         # JSON
goaccess /var/log/nginx/access.log         # web 분석
mtail / fluent-bit                          # local agent
```

```bash
# 자주 사용
journalctl -u nginx --since today | grep -c '500 '
awk '$9==500 {print $1}' access.log | sort | uniq -c | sort -rn | head
```

---

## 12. 보안 로그 — auth / sudo / ssh

```bash
# 실패한 login
journalctl -u sshd | grep 'Failed password'
last -f /var/log/wtmp                      # 성공 login
lastb -f /var/log/btmp                      # 실패 login (root)

# sudo 사용
journalctl _COMM=sudo
ausearch -m USER_CMD
```

---

## 13. 컨테이너 / K8s

- container 의 stdout/stderr → container runtime
- K8s 의 kubelet 이 노드의 `/var/log/pods/`
- **DaemonSet** (Fluentd / Filebeat) 로 외부 시스템 전송

```bash
kubectl logs <pod>
kubectl logs -f <pod>
kubectl logs --previous <pod>              # 이전 컨테이너
docker logs <c>
crictl logs <c>
```

---

## 14. 함정

### 14.1 journald 의 volatile
`/var/log/journal/` 없으면 RAM 만 — reboot 시 사라짐.

### 14.2 디스크 풀
journal / syslog 가 디스크 폭주. retention 정책.

### 14.3 log + secret
패스워드 / token 가 로그에. masking / scrub.

### 14.4 syslog UDP loss
중요 log = TCP / reliable.

### 14.5 timezone
log time = UTC vs local. 표준 UTC 권장.

### 14.6 grep + huge log
`zgrep` for compressed. `ag` / `ripgrep` 더 빠름.

### 14.7 응용이 자체 file rotate
multi-process write race. logrotate copy-truncate 또는 USR1 신호.

### 14.8 K8s 의 log driver
log 가 매우 클 때 backpressure → 응용 stall. async / buffered.

---

## 15. 학습 자료

- **systemd-journald(8)** / **journalctl(1)**
- **rsyslog.conf(5)**
- **The Practice of System and Network Administration**
- **Loki / Fluentd** docs

---

## 16. 관련

- [[systemd]]
- [[monitoring]]
- [[../security/security]]
- [[linux]] — Linux hub
