---
title: "Linux 파일 권한 실무"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T16:30:00+09:00
tags:
  - operating-system
  - linux
  - permissions
---

# Linux 파일 권한 실무

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 실무 / 자주 쓰는 케이스 |

**[[linux|↑ Linux hub]]**

---

개념은 → [[../security/permissions]]
여기는 **실무 / cheatsheet** 만.

---

## 1. ls -l 해석

```
-rwxr-xr-x  1 alice dev   1024 Jan  1 10:00 file
drwxr-xr-x  2 alice dev   4096 Jan  1 10:00 dir
lrwxrwxrwx  1 alice dev      5 Jan  1 10:00 link -> file
```

```
type:
  -  파일
  d  디렉토리
  l  symlink
  c  char device
  b  block device
  s  socket
  p  pipe (FIFO)

mode: rwxr-xr-x
links: 1                   (디렉토리는 . 과 자식 + 1)
owner: alice
group: dev
size:  1024
mtime: Jan 1 10:00
name:  file
```

---

## 2. chmod cheatsheet

```bash
chmod 755 file              # owner rwx, others rx
chmod 644 file              # owner rw, others r
chmod 600 secret            # owner only
chmod 700 ~/.ssh
chmod 644 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_rsa

# 심볼릭
chmod u+x file              # owner add x
chmod g-w file
chmod o= file               # other 0
chmod a+r file              # all read

chmod -R 755 dir/            # 재귀
find . -type d -exec chmod 755 {} +
find . -type f -exec chmod 644 {} +
```

---

## 3. 보통의 권한

| 객체 | mode |
| --- | --- |
| 일반 파일 | 644 |
| 실행 파일 | 755 |
| 디렉토리 | 755 |
| private 파일 | 600 |
| private 디렉토리 | 700 |
| ssh keys | 600 (private), 644 (public) |
| ~/.ssh | 700 |
| ~/.ssh/authorized_keys | 600 |
| /etc/shadow | 640 root:shadow |
| sudo (sticky) | 4755 (setuid root) |
| /tmp (sticky) | 1777 |
| log files | 640 ~ 644 |

---

## 4. chown

```bash
chown alice file
chown alice:dev file
chown -R alice:dev /var/www
chown :dev file              # group 만
```

root 만 chown 가능 (Linux).

---

## 5. umask

```bash
umask                        # 0022 → 새 파일 644 / 새 dir 755
umask 077                    # 600 / 700 (private)
umask 002                    # 664 / 775 (group write)

# 영구
echo "umask 022" >> ~/.bashrc
```

---

## 6. setuid / setgid / sticky

```bash
chmod u+s file               # setuid
chmod g+s file               # setgid
chmod +t dir                 # sticky

chmod 4755 ./prog            # setuid (4000 + 755)
chmod 2755 ./shared_dir       # setgid
chmod 1777 /tmp               # sticky
```

### 6.1 setgid 디렉토리
새 파일이 디렉토리의 group 상속 → 공유 디렉토리에 유용.

```bash
mkdir /shared
chgrp dev /shared
chmod 2775 /shared
# /shared 안 새 파일 = group dev
```

---

## 7. ACL (POSIX ACL)

```bash
# 활성 — ext4/xfs/btrfs 는 mount 기본
mount | grep acl

setfacl -m u:bob:rw file
setfacl -m g:dev:r file
setfacl -m m:rw file              # mask
setfacl -x u:bob file              # 제거
setfacl -b file                    # 모두 제거
setfacl -d -m u:bob:rw dir         # default (자식 상속)

getfacl file

# 백업 / 복원
getfacl -R . > acl.bk
setfacl --restore=acl.bk
```

ls -l 의 `+` 표시:
```
-rw-rw-r--+ 1 alice dev ... file
```

---

## 8. capability (file)

```bash
sudo setcap cap_net_bind_service=+ep /usr/local/bin/myapp
getcap /usr/local/bin/myapp

# 모두 보기
sudo getcap -r / 2>/dev/null
```

자세히 → [[../security/capabilities]]

---

## 9. attr (chattr)

```bash
chattr +i file               # immutable — root 도 변경 X
chattr +a file               # append only
chattr -i file               # 풀기
lsattr file
```

| flag | 의미 |
| --- | --- |
| `i` | immutable |
| `a` | append-only |
| `c` | compressed (Btrfs) |
| `d` | dump 제외 |
| `s` | 안전 삭제 (옛) |
| `j` | data journaling (ext3/4) |

system file 보호 / 로그 보호.

---

## 10. SELinux label

```bash
ls -Z file
chcon -t httpd_sys_content_t /var/www/file
restorecon /var/www/file
```

자세히 → [[../security/selinux-apparmor]]

---

## 11. xattr (extended attribute)

```bash
setfattr -n user.comment -v "important" file
getfattr -d file
attr -l file

# capability / SELinux / NFSv4 ACL 등이 xattr
```

---

## 12. 보안 체크

```bash
# setuid binaries
find / -perm /4000 -type f 2>/dev/null

# world-writable
find / -perm -0002 -type f -not -path '/proc/*' 2>/dev/null

# orphan (no owner)
find / -nouser -o -nogroup 2>/dev/null

# 다른 사용자 writable + setuid
find / -perm -4002 -type f 2>/dev/null
```

---

## 13. 자주 보는 문제

### 13.1 "Permission denied" on script

```bash
chmod +x script.sh
./script.sh
```

### 13.2 ssh 가 키 거부

```
WARNING: UNPROTECTED PRIVATE KEY FILE!
```

```bash
chmod 600 ~/.ssh/id_rsa
chmod 700 ~/.ssh
chmod 644 ~/.ssh/authorized_keys
```

### 13.3 web 서버가 파일 접근 X

```bash
# nginx user (www-data) 가 read
sudo chown -R www-data:www-data /var/www
# 또는
sudo chmod -R o+r /var/www
```

### 13.4 group 멤버 추가 후

```bash
sudo usermod -aG docker $USER
# 새 login / `newgrp docker` 또는 reboot
```

### 13.5 정기 백업의 권한 보존

```bash
cp -a src/ dst/             # archive — 권한 / atime / owner 보존
rsync -aHAX src/ dst/        # ACL / xattr 까지
tar cpf bk.tar dir            # -p preserve
```

---

## 14. 디렉토리 권한의 X 의 의미

```
r — ls 가능
w — 안에 파일 생성 / 삭제 / 이름 변경
x — cd / path traverse / stat
```

→ x 없으면 cd 도 안 됨. ls 도 (실제 metadata 가져오기 안 됨).

---

## 15. 함정

### 15.1 0777 사용
보안 X. 정확한 mode.

### 15.2 ~/.ssh 광범위
ssh 거부.

### 15.3 setuid 남용
권한 escalation. capability 로 대체.

### 15.4 chown -R 의 symlink
`-h` 또는 `-P` 옵션. 기본은 follow.

### 15.5 ACL 의 mask
`getfacl` 의 mask 가 effective 권한 제한.

### 15.6 mount option noacl / nosuid / nodev
필요 / 위험 검토. USB / 외부 = `nosuid,nodev` 권장.

### 15.7 immutable 파일 backup
`tar` / `rsync` 시 `chattr -i` 후. 또는 도구가 무시.

### 15.8 rm 의 권한
파일이 아니라 **디렉토리의 w + x** 가 필요. 읽기 전용 파일도 디렉토리 권한 있으면 rm.

---

## 16. 학습 자료

- **The Linux Programming Interface** Ch. 15
- `man 2 chmod` / `man 7 inode`
- **CIS Linux Benchmark**

---

## 17. 관련

- [[../security/permissions]]
- [[../security/capabilities]]
- [[../filesystem/inode-dentry]]
- [[linux]] — Linux hub
