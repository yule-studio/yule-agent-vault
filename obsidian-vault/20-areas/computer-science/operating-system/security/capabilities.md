---
title: "Linux Capabilities — root 권한 분리"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T15:40:00+09:00
tags:
  - operating-system
  - security
  - capability
---

# Linux Capabilities

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | capability 41개 + 적용 |

**[[security|↑ Security hub]]**

---

## 1. 한 줄

전통 UNIX: **root (UID 0) = 모든 권한**.
Linux capability: 41 개로 세분화 — 응용에 **정확히 필요한 권한만** 부여.

POSIX.1e draft (1998), Linux 2.2+.

---

## 2. 주요 Capability

| Capability | 의미 |
| --- | --- |
| `CAP_AUDIT_CONTROL` | audit |
| `CAP_AUDIT_WRITE` | audit log |
| `CAP_BPF` | eBPF |
| `CAP_CHOWN` | chown |
| `CAP_DAC_OVERRIDE` | mode bits 무시 |
| `CAP_DAC_READ_SEARCH` | read 권한 무시 (read 만) |
| `CAP_FOWNER` | 다른 owner 의 파일 |
| `CAP_FSETID` | setuid bit 유지 |
| `CAP_IPC_LOCK` | mlock 무제한 |
| `CAP_KILL` | 다른 사용자 process 시그널 |
| `CAP_MKNOD` | mknod |
| `CAP_NET_ADMIN` | iptables, network 인터페이스 |
| `CAP_NET_BIND_SERVICE` | port < 1024 bind |
| `CAP_NET_RAW` | raw socket / ping |
| `CAP_PERFMON` | perf |
| `CAP_SETGID / CAP_SETUID` | gid / uid 변경 |
| `CAP_SYS_ADMIN` | "가장 위험" — mount, swap, namespace 등 |
| `CAP_SYS_BOOT` | reboot |
| `CAP_SYS_CHROOT` | chroot |
| `CAP_SYS_MODULE` | kernel module |
| `CAP_SYS_NICE` | nice / 우선순위 |
| `CAP_SYS_PTRACE` | ptrace |
| `CAP_SYS_RESOURCE` | rlimit 초과 |
| `CAP_SYS_TIME` | system time |
| `CAP_SYSLOG` | dmesg |

`man 7 capabilities` 전체 목록.

---

## 3. CAP_SYS_ADMIN — "the new root"

가장 광범위. mount, namespace, sethostname, BPF, ... → 거의 root 와 동등.

→ 가능한 다른 specific capability 로 분리 권장.

---

## 4. process 의 5 세트

```
1. Permitted    가질 수 있는 권한
2. Effective    현재 활성 (실제 권한 검사)
3. Inheritable  exec 후 자식의 Permitted 가 될 수 있는 후보
4. Bounding     상한 — 이걸 넘는 권한 획득 X
5. Ambient (4.3+) Inheritable 의 자동 활성 (normal user 도 exec 후 유지)
```

```bash
# 현재 process
cat /proc/$$/status | grep ^Cap

# 디코드
capsh --decode=000001ffffffffff
```

---

## 5. File Capability

setuid 대신 — 파일에 capability 부여:

```bash
# 부여
sudo setcap cap_net_bind_service=+ep /usr/local/bin/myserver

# 확인
getcap /usr/local/bin/myserver
# /usr/local/bin/myserver = cap_net_bind_service+ep

# 제거
sudo setcap -r /usr/local/bin/myserver
```

xattr 의 `security.capability` 로 저장.

### 5.1 +ep 의 의미

| Flag | 의미 |
| --- | --- |
| `e` | effective (즉시 활성) |
| `p` | permitted |
| `i` | inheritable |

`cap+ep` = 시작 시 effective. setuid 동등 효과를 한 cap 만으로.

---

## 6. setuid 대체 예

```
옛: ping 이 setuid root → raw socket
새: ping 이 CAP_NET_RAW file capability

옛: nginx 가 root start → port 80 bind 후 drop
새: nginx binary 에 CAP_NET_BIND_SERVICE
```

→ exploit 시 권한 한정.

---

## 7. capsh / setpriv

```bash
# capability set 으로 명령 실행
sudo capsh --caps="cap_net_admin,cap_net_raw=eip" -- -c "iptables -L"

# setpriv (util-linux)
sudo setpriv --reuid=nobody --regid=nobody --clear-groups \
             --inh-caps=-all --bounding-set=cap_net_bind_service \
             ./myserver
```

---

## 8. systemd unit

```ini
[Service]
ExecStart=/usr/bin/myapp
AmbientCapabilities=CAP_NET_BIND_SERVICE
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
NoNewPrivileges=yes
User=app
```

→ root 없이 port 80 bind 가능.

---

## 9. Docker / Kubernetes

```bash
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE ...
```

```yaml
securityContext:
  capabilities:
    drop: ["ALL"]
    add: ["NET_BIND_SERVICE"]
```

→ container 기본 capability set 제한.

기본 Docker container 는 약 14 capability — drop ALL + 필요 것만 add 권장.

---

## 10. user namespace + capability

user ns 안의 root = 호스트의 일반 user. capability 도 user ns 안에서 의미.

→ rootless container 의 토대. ns 밖에선 그 capability 무력.

자세히 → [[../virtualization/namespace#10-user-namespace]]

---

## 11. no-new-privileges

```c
prctl(PR_SET_NO_NEW_PRIVS, 1);
```

```ini
# systemd
NoNewPrivileges=yes
```

```yaml
# K8s
securityContext:
  allowPrivilegeEscalation: false
```

→ 이후 exec 으로 capability 획득 X (setuid binary 등 무력).

---

## 12. ptrace + capability

ptrace 는 같은 user 또는 CAP_SYS_PTRACE. yama 정책으로 추가 제한:

```bash
cat /proc/sys/kernel/yama/ptrace_scope
# 0 same user OK
# 1 parent only
# 2 admin only (CAP_SYS_PTRACE)
# 3 no ptrace
```

---

## 13. CAP_BPF / CAP_PERFMON (5.8+)

eBPF / perf 의 권한을 SYS_ADMIN 에서 분리. observability 도구를 더 적은 권한으로.

---

## 14. 함정

### 14.1 CAP_SYS_ADMIN 남용
"new root" — 다른 specific cap 으로 분리.

### 14.2 setuid binary 유지
file capability 로 마이그.

### 14.3 ambient 누락
Inheritable 만 설정 → exec 후 effective X. ambient 또는 file cap.

### 14.4 user ns 에서의 cap 환상
호스트에선 의미 X.

### 14.5 container 의 cap-drop=ALL 후 동작 안 함
응용에 필요한 cap 확인 후 add.

### 14.6 CAP_NET_ADMIN + iptables
정책 변경 가능 → 거의 root 동등.

### 14.7 PR_SET_NO_NEW_PRIVS 의 의존
seccomp / capability 조합으로 sandbox 강화.

---

## 15. 학습 자료

- `man 7 capabilities`
- **The Linux Programming Interface** Ch. 39
- **Capabilities — A POSIX.1e Brief**

---

## 16. 관련

- [[permissions]]
- [[seccomp]]
- [[sandbox]]
- [[../virtualization/namespace]]
- [[security]] — Security hub
