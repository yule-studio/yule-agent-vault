---
title: "File permissions — chmod / chown / umask / setuid"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T06:40:00+09:00
tags: [devops, linux, permissions]
---

# File permissions — chmod / chown / umask / setuid

**[[linux|↑ linux]]**

---

## 1. 권한 모델

```
$ ls -l file
-rw-r--r-- 1 alice users 1024 May 15 10:00 file
 ↑│  │  │  │  ↑     ↑
 │└──┴──┴──┴─ 권한 (user / group / others)
 │             │  │  │
 │             │  │  └─ others (r--)
 │             │  └──── group  (r--)
 │             └─────── user   (rw-)
 │
 └─ 종류: -=파일, d=디렉터리, l=링크, c/b=device, s=socket
```

| | r (4) | w (2) | x (1) |
| --- | --- | --- | --- |
| 파일 | 읽기 | 쓰기 | 실행 |
| 디렉터리 | ls | 파일 추가/삭제 | cd / access |

---

## 2. chmod

```bash
# 숫자 (octal)
chmod 755 script.sh       # rwxr-xr-x
chmod 644 file.txt        # rw-r--r--
chmod 600 ~/.ssh/id_rsa   # rw-------
chmod 700 ~/.ssh          # rwx------

# 기호
chmod u+x script.sh       # user 에 실행 추가
chmod g-w file            # group 에서 쓰기 제거
chmod a+r file            # all 에 읽기
chmod o= file             # others 모든 권한 제거
chmod ug=rw file          # user/group 을 rw 로

# 재귀
chmod -R 755 /opt/app
```

---

## 3. chown / chgrp

```bash
chown alice file
chown alice:staff file
chown :staff file               # group 만
chgrp users file

chown -R alice:alice /home/alice

# numeric ID
chown 1000:1000 file
```

---

## 4. umask (default 권한)

```bash
$ umask
0022

# 새 파일 권한 = 666 - umask = 644 (rw-r--r--)
# 새 디렉터리 = 777 - umask = 755 (rwxr-xr-x)

umask 077                       # private (rw-------)
umask 002                       # group 공유 (rw-rw-r--)
```

→ `/etc/profile` 또는 `~/.bashrc` 에 설정.

---

## 5. setuid / setgid / sticky bit

```bash
# setuid (4xxx) — 실행 시 user owner 권한
$ ls -l /usr/bin/passwd
-rwsr-xr-x 1 root root ... /usr/bin/passwd
   ↑ s = setuid

# setgid (2xxx) — 디렉터리에 적용 시 new file 의 group 이 부모 group
chmod g+s /shared/dir

# sticky (1xxx) — 파일 owner 만 삭제 가능
$ ls -ld /tmp
drwxrwxrwt 11 root root ... /tmp
         ↑ t = sticky

chmod +t /shared
chmod 1777 /shared
```

→ **setuid 는 보안 위험**. 신중히.

---

## 6. ACL (확장 권한)

```bash
# 기본 user/group/other 외에 추가 user/group 권한
setfacl -m u:bob:rx file        # bob 에 r+x 추가
setfacl -m g:devs:rwx /opt      # devs group 에 rwx
getfacl file
setfacl -x u:bob file           # 제거
setfacl -b file                 # 전체 ACL 제거

# default ACL — 디렉터리의 새 파일 default
setfacl -d -m g:devs:rwx /opt
```

---

## 7. 특수 디렉터리 권한

```bash
~/.ssh                700 (rwx------)
~/.ssh/id_rsa         600 (rw-------)   # private key
~/.ssh/id_rsa.pub     644 (rw-r--r--)
~/.ssh/authorized_keys  600
~/.ssh/known_hosts    644

/var/log              755 (root:root, 일부 group root 쓰기)
/etc/shadow           640 (root:shadow) — 패스워드 hash
/etc/passwd           644
/tmp                  1777 (sticky)
```

---

## 8. Linux capabilities (sudo 대안)

```bash
# 특정 syscall 만 허용 — full root 권한 X
setcap 'cap_net_bind_service=+ep' /usr/local/bin/nginx
# → nginx 가 80 / 443 port bind 가능 (root 아니어도)

getcap /usr/local/bin/nginx
```

→ Docker 의 `--cap-add` 와 동일 개념.

---

## 9. SELinux / AppArmor (MAC)

DAC (discretionary access control) + MAC (mandatory):
- **SELinux** — Red Hat / CentOS 계열
- **AppArmor** — Ubuntu / Debian
- 정책 위반 시 권한 있어도 deny.

```bash
# SELinux
sestatus
ls -Z file                    # context
setenforce 0                  # permissive (debug)
chcon -t httpd_sys_content_t file

# AppArmor
aa-status
aa-complain /usr/sbin/nginx
```

---

## 10. 함정

1. **`chmod 777`** — 모두 쓰기 / 실행. 보안 재앙.
2. **`~/.ssh` 권한 너무 느슨** — ssh client 가 거부.
3. **setuid 남발** — 권한 상승 취약.
4. **umask 가 너무 느슨** — 새 파일이 group-readable → 정보 누출.
5. **root 로 모두 소유** — 일반 user 가 write 못 함.
6. **사라진 sticky bit** — `/tmp` 가 1777 아님 → 다른 사용자가 파일 삭제 가능.

---

## 11. 관련

- [[linux|↑ linux]]
- [[users-and-groups]]
- [[ssh]]
- [[../docker/security|↗ docker security]]
