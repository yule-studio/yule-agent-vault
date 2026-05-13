---
title: "SELinux / AppArmor — MAC (Mandatory Access Control)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T15:45:00+09:00
tags:
  - operating-system
  - security
  - selinux
  - apparmor
---

# SELinux / AppArmor

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | MAC 비교 + 사용 |

**[[security|↑ Security hub]]**

---

## 1. 한 줄

DAC (mode bits / ACL) 위에 추가되는 **시스템 정책 강제**. 응용이 owner 라도 정책이 막으면 못 함.

- **SELinux** — NSA + Red Hat. label 기반. 매우 강력 / 복잡.
- **AppArmor** — Canonical / SUSE. path 기반. 단순.

LSM (Linux Security Module) 프레임워크 위.

---

## 2. SELinux

### 2.1 모드

```bash
getenforce
# Enforcing | Permissive | Disabled

sudo setenforce 0       # permissive (디버그)
sudo setenforce 1       # enforcing

# /etc/selinux/config
SELINUX=enforcing
SELINUXTYPE=targeted
```

### 2.2 Label / Context

모든 파일 / 프로세스에 label:
```
user:role:type:level
system_u:object_r:httpd_sys_content_t:s0
```

```bash
ls -Z /var/www/html
# unconfined_u:object_r:httpd_sys_content_t:s0 index.html

ps -eZ | grep httpd
# system_u:system_r:httpd_t:s0 ... /usr/sbin/httpd
```

→ 정책: `httpd_t` 프로세스가 `httpd_sys_content_t` 만 read.

### 2.3 잘못된 label 수정

```bash
restorecon -v /var/www/html/index.html
chcon -t httpd_sys_content_t /var/www/html/index.html
```

### 2.4 디버그

```bash
sealert -a /var/log/audit/audit.log
ausearch -m AVC -ts recent
```

AVC (Access Vector Cache) denial 로그.

### 2.5 boolean

```bash
getsebool -a | grep httpd
setsebool -P httpd_can_network_connect 1
```

정책의 on/off switch.

### 2.6 module
```bash
semodule -l
semodule -i custom.pp
```

---

## 3. AppArmor

### 3.1 모드

```bash
sudo aa-status
sudo aa-enforce /etc/apparmor.d/usr.sbin.nginx
sudo aa-complain /etc/apparmor.d/usr.sbin.nginx
sudo aa-disable /etc/apparmor.d/usr.sbin.nginx
```

### 3.2 Profile (path 기반)

```
/etc/apparmor.d/usr.sbin.nginx:

#include <tunables/global>
profile nginx /usr/sbin/nginx {
    #include <abstractions/base>
    #include <abstractions/nameservice>

    capability net_bind_service,
    capability setuid,
    capability setgid,

    /usr/sbin/nginx mr,
    /etc/nginx/** r,
    /var/log/nginx/*.log w,
    /var/www/** r,
    /var/run/nginx.pid w,

    network inet stream,
    network inet6 stream,

    /tmp/** rwk,           # 일부 권한
    deny /etc/shadow r,
}
```

→ path 단위 권한. 직관적.

### 3.3 디버그

```bash
dmesg | grep -i apparmor
journalctl -k --grep apparmor
```

`audit: type=1400 ... apparmor="DENIED"` 로그.

---

## 4. SELinux vs AppArmor

| | SELinux | AppArmor |
| --- | --- | --- |
| 모델 | label (type enforcement) | path 기반 |
| 강력함 | ★★★★★ | ★★★ |
| 복잡도 | 매우 높음 | 보통 |
| 배포판 | RHEL/Fedora/CentOS | Ubuntu/SUSE |
| inheritance | 정책 그래프 | 단순 |
| 디버그 | sealert, ausearch | dmesg |
| 학습 곡선 | 가파름 | 완만 |

→ 강한 보안 요구 (정부, 금융) = SELinux. 일반 운영 = AppArmor 도 충분.

---

## 5. Common Audit Messages

```bash
type=AVC msg=audit(...): avc:  denied  { read } for  pid=... comm="nginx" \
    name="passwd" dev="dm-0" ino=... \
    scontext=system_u:system_r:httpd_t:s0 \
    tcontext=system_u:object_r:passwd_file_t:s0 \
    tclass=file
```

→ httpd 가 /etc/passwd read 시도 → 차단.

---

## 6. Container + MAC

Docker / K8s 의 기본 profile:
- Docker: `docker-default` (AppArmor) 또는 `container_t` (SELinux)
- runc / containerd 가 자동 적용

```bash
docker run --security-opt apparmor=docker-default ...
docker run --security-opt label=type:container_t ...
```

container escape 의 추가 방어선.

---

## 7. Permissive 모드

```bash
# SELinux
sudo setenforce 0

# AppArmor
sudo aa-complain /etc/apparmor.d/profile
```

차단 X but log 만 — 정책 디버그 / 마이그레이션 시.

⚠️ 운영은 enforcing.

---

## 8. 정책 작성 도구

### 8.1 SELinux
```bash
# audit2allow — denial log 에서 rule 자동 생성
ausearch -m AVC | audit2allow -M mypolicy
semodule -i mypolicy.pp
```

### 8.2 AppArmor

```bash
sudo aa-genprof /path/to/binary       # 대화형 학습
sudo aa-logprof                        # log 기반 갱신
```

→ 응용 실행 → 접근 시도를 학습 → profile 생성.

---

## 9. systemd + MAC

```ini
[Service]
ExecStart=/usr/bin/myapp
# AppArmor
AppArmorProfile=myapp

# SELinux
SELinuxContext=system_u:system_r:myapp_t:s0
```

---

## 10. 비활성 (디버그 / 임시)

⚠️ 권장 X. 정책 수정이 정답.

```bash
# SELinux 일시
sudo setenforce 0

# 영구 (재부팅 필요)
# /etc/selinux/config
SELINUX=disabled

# AppArmor
sudo systemctl stop apparmor
```

비활성 후 다시 enforcing 시 label 재구성 필요할 수 있음.

---

## 11. K8s 의 MAC

| | 의미 |
| --- | --- |
| Pod Security Standards | privileged / baseline / restricted |
| Pod Security Admission | enforce policy |
| seccomp / AppArmor / SELinux | pod spec |

```yaml
securityContext:
  seLinuxOptions:
    level: "s0:c123,c456"
  appArmorProfile:
    type: RuntimeDefault
```

---

## 12. 함정

### 12.1 SELinux disabled 후 enforcing 복귀
label 깨짐 → autorelabel:
```bash
sudo touch /.autorelabel
sudo reboot
```

### 12.2 chcon 후 restorecon
chcon 은 일시 — restorecon (정책 기준) 으로 영구.

### 12.3 path 변경 후 AppArmor
profile 의 path 일치 필요. profile 갱신.

### 12.4 audit log 폭증
정책 잘못 → denial 폭증 → /var/log 폭주.

### 12.5 container + host MAC mismatch
host 의 SELinux denial 이 container 안에선 mystery.

### 12.6 Docker --privileged
대부분 MAC 비활성. 보안 거의 X.

### 12.7 NFS / overlay 의 label
path 기반은 OK, label 기반은 fs / mount option 필요.

---

## 13. 학습 자료

- **SELinux Notebook** — github / 무료 책
- **AppArmor wiki** — gitlab.com/apparmor
- **Red Hat SELinux Guide**
- **Mandatory Access Control**

---

## 14. 관련

- [[permissions]] — DAC
- [[capabilities]]
- [[seccomp]]
- [[sandbox]]
- [[security]] — Security hub
