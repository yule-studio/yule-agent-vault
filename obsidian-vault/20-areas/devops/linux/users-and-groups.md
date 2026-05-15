---
title: "Users / groups / sudo / PAM"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T06:58:00+09:00
tags: [devops, linux, users]
---

# Users / groups / sudo / PAM

**[[linux|↑ linux]]**

---

## 1. user 관리

```bash
# 추가
sudo useradd -m -s /bin/bash alice          # home + shell
sudo useradd -m -G sudo,docker alice
sudo passwd alice

# 정보
id alice                                    # uid, gid, groups
finger alice                                # 정보 (옵션)
cat /etc/passwd | grep alice
groups alice

# 수정
sudo usermod -aG docker alice               # group 추가 (★ -a 중요)
sudo usermod -s /bin/zsh alice              # shell
sudo usermod -L alice                        # lock (login 차단)
sudo usermod -U alice                        # unlock

# 삭제
sudo userdel alice                           # home 유지
sudo userdel -r alice                        # home 까지
```

→ **`usermod -G` 만 쓰면 기존 group 사라짐**. 반드시 `-aG`.

---

## 2. /etc/passwd / /etc/shadow / /etc/group

```
# /etc/passwd
alice:x:1000:1000:Alice,,,:/home/alice:/bin/bash
  │   │  │    │     │       │            │
  │   │  │    │     │       │            └─ shell
  │   │  │    │     │       └─ home
  │   │  │    │     └─ GECOS (info)
  │   │  │    └─ GID
  │   │  └─ UID
  │   └─ password 위치 (x = shadow)
  └─ username

# /etc/shadow (600, root)
alice:$6$salt$hash:19000:0:99999:7:::
  │     │            │   │   │   │
  │     │            │   │   │   └─ warn (만료 전)
  │     │            │   │   └─ max age
  │     │            │   └─ min age
  │     │            └─ last change (days since epoch)
  │     └─ hashed password ($6 = sha512crypt)
  └─ username

# /etc/group
sudo:x:27:alice,bob
```

---

## 3. group

```bash
sudo groupadd devs
sudo groupdel devs
sudo gpasswd -a alice devs                  # add
sudo gpasswd -d alice devs                  # remove

# 임시 group 전환
newgrp devs                                  # 같은 shell 에서 group 변경
```

---

## 4. sudo

```bash
sudo cmd                                     # root 로
sudo -u alice cmd                            # alice 로
sudo -i                                      # interactive root shell
sudo su -                                    # 비슷 (legacy)

sudo -l                                      # 내가 sudo 가능한 명령
```

```bash
# /etc/sudoers 또는 /etc/sudoers.d/
sudo visudo                                  # 안전한 편집기 (syntax check)

# 표준 줄
alice ALL=(ALL:ALL) ALL                     # 전부 sudo OK
%sudo ALL=(ALL:ALL) ALL                     # sudo group 전체

# 비밀번호 없이
alice ALL=(ALL) NOPASSWD: ALL                # 위험
alice ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx  # 특정 명령만

# 안전 옵션
Defaults env_reset
Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Defaults logfile="/var/log/sudo.log"
Defaults timestamp_timeout=15
```

→ **visudo 사용** (직접 편집 시 lock-out 위험).

---

## 5. PAM (Pluggable Authentication Modules)

```
/etc/pam.d/sshd
/etc/pam.d/sudo
/etc/pam.d/login
```

모듈 chain:
```
# /etc/pam.d/sshd 의 일부
auth    required    pam_unix.so      # /etc/shadow 로 인증
account required    pam_unix.so      # account 유효
session required    pam_limits.so    # ulimit
session optional    pam_motd.so      # MOTD 출력
```

→ MFA, LDAP, Kerberos 통합도 PAM 으로.

---

## 6. user 종류

| 종류 | UID | 예 |
| --- | --- | --- |
| **root** | 0 | superuser |
| **system** | 1-999 | nginx, postgres, ssh, ... |
| **regular** | 1000+ | 사람 사용자 |
| **nobody** | 65534 | 비특권 (NFS 등) |

→ service 는 system user (login 불가, shell `/usr/sbin/nologin`).

---

## 7. service user 생성

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin myapp
sudo mkdir -p /var/log/myapp /var/lib/myapp
sudo chown myapp:myapp /var/log/myapp /var/lib/myapp
```

→ systemd service 의 `User=myapp` 으로 실행 (root 아님 ★).

---

## 8. password policy

```bash
# /etc/login.defs
PASS_MAX_DAYS 90
PASS_MIN_DAYS 1
PASS_MIN_LEN 12
PASS_WARN_AGE 7

# 강력한 password (PAM)
# /etc/pam.d/common-password
password requisite pam_pwquality.so retry=3 minlen=12 difok=4 \
    ucredit=-1 lcredit=-1 dcredit=-1 ocredit=-1
```

---

## 9. /etc/skel (new user 기본 file)

```bash
ls -la /etc/skel/
# .bashrc, .profile, .ssh/, ...

# useradd 시 /etc/skel/ 내용이 new home 으로 복사
```

→ 회사 default `.bashrc` 표준화.

---

## 10. 보안 권장

```
- root login disable (PermitRootLogin no)
- 개인 user → sudo
- service 별 user (system, login 불가)
- sudo NOPASSWD 는 정말 필요한 명령만
- /etc/shadow 권한 600
- 정기 password rotation
- 사용 안 하는 user lock (usermod -L)
- 퇴사자 즉시 제거
- audit log (auditd)
```

---

## 11. 함정

1. **`usermod -G`** (without `-a`) — 기존 group 사라짐.
2. **`/etc/sudoers` 직접 편집** — syntax error 시 sudo 전부 lock-out.
3. **NOPASSWD ALL** — 비밀번호 없이 root → 키 탈취 시 재앙.
4. **root 로 daemon 실행** — exploit 시 즉시 RCE.
5. **공유 계정** — 누가 했는지 모름 → audit 불가.
6. **password 만 — MFA 없음** — brute-force / phishing.

---

## 12. 관련

- [[linux|↑ linux]]
- [[file-permissions]]
- [[ssh]]
- [[../security-ops/security-ops|↗ security-ops]]
