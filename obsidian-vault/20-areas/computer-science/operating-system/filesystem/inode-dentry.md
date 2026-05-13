---
title: "Inode / Dentry / Hardlink / Symlink"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T12:35:00+09:00
tags:
  - operating-system
  - filesystem
  - inode
---

# Inode / Dentry

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 메타데이터 + 링크 |

**[[filesystem|↑ Filesystem hub]]**

---

## 1. Inode

파일 1 개당 1 개. **데이터의 모든 메타데이터** 보유:

- 파일 타입 (regular, dir, symlink, socket, ...)
- 권한 (mode)
- UID / GID
- size
- timestamps (atime, mtime, ctime)
- link count (몇 개 이름이 가리키는지)
- 데이터 block 의 위치 (direct / indirect / extent)

⚠️ **이름은 inode 안에 없음** — 디렉토리가 이름 ↔ inode 매핑.

```bash
ls -i file
stat file
```

---

## 2. Dentry — Directory Entry

```
디렉토리 = inode + 데이터 (= 이름 → inode 번호 매핑 표)

/home/alice/file.txt:
  / 의 inode → "home" : inode 100
  /home 의 inode → "alice" : inode 200
  /home/alice 의 inode → "file.txt" : inode 300
```

Linux 커널 안에선 `dentry` cache (DCAche) 로 path lookup 가속.

---

## 3. inode 번호

각 파일 시스템 안에서 unique. 같은 inode 가 같은 파일.

```bash
ls -li
# 12345 -rw-r--r-- 2 me me 100 Jan 1 file.txt
```

`12345` 가 inode 번호.

mount 다르면 inode 충돌 가능 (각 fs 가 독립).

---

## 4. Hard Link

같은 inode 를 가리키는 두 이름.

```bash
echo hello > a.txt
ln a.txt b.txt          # hardlink
ls -li
# 12345 ... 2 a.txt
# 12345 ... 2 b.txt     ← 같은 inode, link count = 2

rm a.txt                 # link count = 1
rm b.txt                 # link count = 0 → 실제 삭제
```

규칙:
- 같은 파일 시스템 안에서만
- 디렉토리는 hardlink 금지 (cycle 위험)
- link count 0 + open FD 0 = 데이터 해제

---

## 5. Symbolic Link (Symlink)

이름이 가리키는 **경로 문자열** 보유. 다른 inode.

```bash
ln -s /etc/hosts hosts
ls -li
# 12345 lrwxrwxrwx 1 me me ... hosts -> /etc/hosts
# 67890 ... /etc/hosts (다른 inode)
```

- 다른 파일 시스템 가능
- 깨질 수 있음 (target 없어짐)
- 디렉토리도 가능
- 권한은 target 기준

```bash
readlink hosts
realpath hosts           # 정규화
```

---

## 6. atime / mtime / ctime

| 시각 | 변경 시점 |
| --- | --- |
| **atime** | 마지막 access |
| **mtime** | 마지막 데이터 modify |
| **ctime** | 마지막 inode 변경 (chmod, rename, link 등) |
| **btime** (XFS, ext4) | 생성 시간 (`stat` 의 `Birth`) |

⚠️ atime = 모든 read 마다 갱신 → 디스크 write 부담. **`noatime` mount** 가 일반 권장.

```bash
mount -o remount,noatime /
# /etc/fstab
/dev/sda1 / ext4 defaults,noatime 0 1
```

---

## 7. 권한 (mode)

```
-rwxr-xr--
─ type (file - / dir d / link l / socket s / pipe p)
 rwx     owner
    r-x  group
       r-- others
```

```bash
chmod 755 file        # rwx r-x r-x
chmod u+x,g-w file
chown alice:dev file
```

### 7.1 특수 비트

- **setuid (s)** — exec 시 owner 권한으로 실행 (`/bin/passwd`)
- **setgid (s)** — exec 시 group 권한 + 디렉토리는 자식 파일 group 상속
- **sticky (t)** — 디렉토리: owner 만 자기 파일 삭제 (`/tmp`)

```bash
chmod 4755 ./prog     # setuid
chmod 1777 /tmp        # sticky
```

자세히 → [[../security/permissions]]

---

## 8. 파일 타입

```c
S_ISREG(mode)       // 일반 파일
S_ISDIR(mode)       // 디렉토리
S_ISLNK(mode)       // symlink
S_ISCHR(mode)       // character device
S_ISBLK(mode)       // block device
S_ISFIFO(mode)      // FIFO / pipe
S_ISSOCK(mode)      // Unix domain socket
```

---

## 9. File Descriptor (FD)

프로세스가 open() 한 결과. 그 프로세스에서만 의미 있는 정수.

```
프로세스의 FD table → kernel 의 open file table → inode

FD 0 = stdin
FD 1 = stdout
FD 2 = stderr
```

자세히 → [[../process/pcb#5-files_struct--fd-table]]

---

## 10. open file count

```bash
ulimit -n                            # 프로세스 당 (소프트)
ulimit -Hn                           # 하드
cat /proc/sys/fs/file-max            # 시스템 전체

# 현재 사용
cat /proc/sys/fs/file-nr
lsof -p $PID | wc -l
```

서버는 보통 65535+. 부족 시 `Too many open files`.

---

## 11. inode 고갈

inode 수는 **파일 시스템 만들 때 고정** (ext4 기본 ratio).

```bash
df -i                                 # inode 사용량
# /dev/sda1   100M  100M  0  100%

# 작은 파일 폭증 시
```

해결: 큰 파일 시스템 / `mkfs.ext4 -i` 로 ratio 조정.

---

## 12. 디렉토리 lookup 효율

```
ls 같은 디렉토리에 수만 파일 → 디렉토리 read 가 slow (O(N))
ext4: htree (B-tree-like) → O(log N)
XFS: B-tree
```

`dir_index` mount option (ext4) 활성.

---

## 13. 파일 삭제

`unlink()` — 디렉토리에서 이름 제거 + inode link count 감소.
0 + 어떤 process 도 open X = 데이터 해제.

```c
unlink("/tmp/file");        // 이름 제거
close(fd);                   // 모든 FD 닫힘
// 데이터 해제
```

`mv` 는 같은 fs 면 rename (inode 그대로), 다른 fs 면 copy + delete.

---

## 14. 파일 일관성

- `rename(old, new)` — atomic (같은 fs)
- `link()` + `unlink()` — atomic 으로 atomicaly swap

원자 update 패턴:
```
write tmp.txt
fsync tmp.txt
rename tmp.txt target.txt
fsync 디렉토리 (target 의 부모)
```

자세히 → [[fsync-durability]]

---

## 15. dentry cache (DCAche)

커널이 path 의 dentry 를 RAM 에 캐시:

```bash
cat /proc/meminfo | grep KReclaimable
slabtop | grep dentry
```

대량 stat 후 dentry 누적 → memory pressure.

---

## 16. extended attribute (xattr)

inode 의 추가 key-value 메타:

```bash
setfattr -n user.comment -v "important" file
getfattr -d file
# 또는 attr / chattr
```

SELinux label, NFS ACL, capability, checksum 등에 사용.

---

## 17. immutable / append-only

```bash
chattr +i file       # immutable — root 도 변경 X
chattr +a file       # append-only
lsattr file
```

log 보호 / system files.

---

## 18. 함정

### 18.1 atime 갱신 비용
`noatime` 권장.

### 18.2 hardlink 의 cross-fs 시도
실패. symlink 사용.

### 18.3 broken symlink
target 사라짐 → ENOENT.

### 18.4 link count > 0 이지만 open 후 unlink
삭제는 close 시. `lsof | grep deleted` 로 디스크 누수 진단.

### 18.5 dentry cache 폭증
sync ; echo 2 > /proc/sys/vm/drop_caches.

### 18.6 ext4 dir_index off + 거대 디렉토리
slow. 활성.

### 18.7 inode 고갈
df -i 로 확인. mkfs 의 `-i` ratio.

### 18.8 setuid 보안 위험
root 권한 escalation 의 흔한 통로.

---

## 19. 학습 자료

- **OSTEP** Ch. 39-40
- **The Linux Programming Interface** Ch. 14-15, 18
- `man 2 stat`, `man 7 inode`

---

## 20. 관련

- [[journaling]]
- [[../security/permissions]]
- [[filesystem]] — FS hub
