---
title: "UNIX 권한 — UID / GID / mode / ACL / setuid"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T15:35:00+09:00
tags:
  - operating-system
  - security
  - permissions
---

# UNIX 권한

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | mode / setuid / ACL |

**[[security|↑ Security hub]]**

---

## 1. UID / GID

각 프로세스 / 파일에 소유자 (UID) + 그룹 (GID).

- UID 0 = root (전통적 슈퍼유저)
- 일반 user UID ≥ 1000
- 시스템 user UID 1-999

```bash
id
# uid=1000(alice) gid=1000(alice) groups=1000(alice),4(adm),27(sudo)

cat /etc/passwd | head -3
cat /etc/group | head -3
```

---

## 2. process 의 UID/GID

- **Real UID** — 시작 사용자
- **Effective UID** — 권한 검사용
- **Saved UID** — setuid 복원용
- **Filesystem UID** — 파일 접근 (Linux 특)

```c
getuid(), geteuid(), getresuid(...)
setuid(), seteuid(), setresuid(...)
```

setuid binary 가 effective 만 root 로 바꿔 권한 escalation.

---

## 3. 파일 권한 — Mode

```
-rwxr-xr--
─type rwx r-x r--
       owner group other
```

비트로:
```
4 r
2 w
1 x

owner=7 (rwx) group=5 (r-x) other=4 (r--) → 754
```

```bash
chmod 755 file
chmod u+x file
chmod -R go-w /path
```

---

## 4. 특수 비트

| 비트 | 값 | 의미 |
| --- | --- | --- |
| **setuid** | 4000 | exec 시 effective UID = owner |
| **setgid** | 2000 | exec 시 effective GID = group |
| **sticky** | 1000 | dir: owner 만 자기 파일 삭제 |

```bash
chmod 4755 ./prog       # setuid (s)
chmod 1777 /tmp          # sticky (t)
chmod 2755 ./dir         # setgid (s)
```

### 4.1 setuid binary 예
```bash
ls -l /usr/bin/passwd
# -rwsr-xr-x 1 root root ... /usr/bin/passwd
```

passwd 는 root 만 쓸 수 있는 `/etc/shadow` 갱신 — setuid root.

⚠️ setuid = exploit 의 흔한 표적. 최소화.

---

## 5. 디렉토리 권한

```
r — 디렉토리 목록 (ls)
w — 안의 파일 생성 / 삭제
x — 디렉토리 안 접근 (path traverse)
```

```
755 /path             # 모두 ls, owner write
700 ~/.ssh             # owner 만
1777 /tmp              # 누구나 쓰지만 sticky
```

---

## 6. umask

새 파일의 기본 권한.

```bash
umask                  # 0022
umask 077              # private (700 / 600)
```

```
default file:  666 & ~umask = 644
default dir:   777 & ~umask = 755
```

systemd 의 daemon 은 보통 0022 또는 자체 설정.

---

## 7. chown / chgrp

```bash
chown alice:dev file
chown -R alice /home/alice
chgrp dev file
```

root 만 chown 가능 (Linux). chgrp 는 owner + 그룹 멤버.

---

## 8. ACL (POSIX ACL)

mode bits 보다 세밀:

```bash
setfacl -m u:bob:rw file              # bob 에게 read+write
setfacl -m g:dev:r file
getfacl file

# default ACL (디렉토리)
setfacl -d -m u:bob:rw dir
```

ext4 / XFS / Btrfs 지원. mount option `acl`.

→ ACL 적용 시 mode bits 의 `+` 표시.

---

## 9. Extended Attribute (xattr)

```bash
setfattr -n user.author -v "alice" file
getfattr -d file

# 시스템 xattr
# security.selinux
# security.capability
# trusted.*
```

ACL, SELinux label, capability 등이 xattr 로 저장.

---

## 10. 그룹 관리

```bash
groupadd dev
useradd -G dev,sudo alice
gpasswd -a alice dev
usermod -aG docker alice
groups alice
```

`/etc/group`.

---

## 11. /etc/passwd / /etc/shadow

```
/etc/passwd:
  alice:x:1000:1000:Alice:/home/alice:/bin/bash

/etc/shadow:
  alice:$6$salt$hash:19000:0:99999:7:::
```

| 필드 | 의미 |
| --- | --- |
| name | 사용자 |
| `x` | password placeholder (실제는 shadow) |
| uid | UID |
| gid | primary GID |
| GECOS | full name 등 |
| home | $HOME |
| shell | 기본 shell |

`shadow`:
- hash (보통 `$6$` = sha512, 또는 `$y$` yescrypt)
- 마지막 변경일 / 만료 정책

---

## 12. PAM

자세히 → [[security#10-pam-pluggable-authentication-modules]]

---

## 13. Linux 의 추가

### 13.1 file capability (xattr)
```bash
setcap cap_net_bind_service+ep /usr/bin/myserver
getcap /usr/bin/myserver
```

→ setuid 없이 특정 capability 만. 자세히 → [[capabilities]]

### 13.2 immutable / append-only
```bash
chattr +i file              # immutable — root 도 수정 X
chattr +a logfile           # append only
lsattr file
```

### 13.3 user_namespace
ns 안의 root = 호스트의 일반 user. rootless container.
자세히 → [[../virtualization/namespace#10-user-namespace]]

---

## 14. 운영 권장

- root 로 응용 실행 X
- 비밀번호 정책 (PAM)
- ssh root login 금지
- sudo (NOPASSWD 신중)
- 정기적 권한 audit
- setuid binary 최소화
- `/tmp` sticky
- ssh key 권한 600 / 700

---

## 15. 보안 체크

```bash
# setuid binary 목록
find / -perm /4000 -type f 2>/dev/null

# world-writable
find / -perm -0002 -type f 2>/dev/null

# 다른 사용자 holdable
find / -uid 0 -perm -0002 -type f 2>/dev/null

# 패키지 검증
debsums -c
rpm -Va
```

---

## 16. 함정

### 16.1 root 로 응용
사고의 ground zero.

### 16.2 setuid binary 위험
exploit 의 표적. capability 로 대체.

### 16.3 chmod 777
보안 X. 정확한 권한.

### 16.4 ~/.ssh 600/700
넓으면 ssh 거부.

### 16.5 sudo NOPASSWD
편리 vs 위험.

### 16.6 group 멤버 추가 후 reload
새 group 적용 = 새 login 또는 `newgrp`.

### 16.7 ACL + mode bits 혼동
ACL 이 우선. `+` 표시 보기.

### 16.8 umask 077 의 영향
서비스 시작 시 mode 600 → 다른 user 접근 X.

---

## 17. 학습 자료

- **The Linux Programming Interface** Ch. 8, 35
- **man 7 credentials**
- **CIS Linux Benchmark**

---

## 18. 관련

- [[capabilities]]
- [[selinux-apparmor]]
- [[../filesystem/inode-dentry]]
- [[security]] — Security hub
