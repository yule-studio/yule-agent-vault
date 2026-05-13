---
title: "Filesystem (Hub)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T12:30:00+09:00
tags:
  - operating-system
  - filesystem
  - hub
---

# Filesystem (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | hub + 6 개 세부 노트 |

**[[../operating-system|↑ OS hub]]**

---

## 1. 한 줄

디스크 (또는 메모리 / 네트워크) 위에 **파일 + 디렉토리** 추상화를 만들어 응용에 제공.

---

## 2. 핵심 추상화

- **File** — 이름 + 데이터 + 메타데이터
- **Directory** — 이름 → 파일 매핑
- **Inode** — 파일의 메타데이터 (소유자, 권한, 크기, 블록 위치)
- **Dentry** — 디렉토리 엔트리 (이름 → inode 번호)
- **Mount** — 파일 시스템을 디렉토리 트리에 부착
- **File Descriptor** — 프로세스가 열어둔 파일의 핸들

자세히 → [[inode-dentry]]

---

## 3. 주요 파일 시스템

| FS | 특징 |
| --- | --- |
| **ext2/3/4** | Linux 기본. journaling (3+), extents (4) |
| **XFS** | 고성능 / 큰 파일 / 큰 디스크 |
| **Btrfs** | COW, snapshot, RAID, 검사 |
| **ZFS** | COW + ARC + zpool — FreeBSD / Solaris |
| **F2FS** | Flash 최적 (SSD) |
| **APFS** | macOS 2017, COW + snapshot |
| **NTFS** | Windows |
| **FAT32 / exFAT** | USB / 호환 |
| **tmpfs** | RAM 위의 파일시스템 |
| **procfs / sysfs / cgroupfs** | 가상 파일시스템 (커널 인터페이스) |
| **NFS / CIFS / SMB / 9P** | 네트워크 |
| **FUSE** | 유저 공간 FS |

자세히 → [[ext-xfs-zfs]]

---

## 4. Journaling

자세히 → [[journaling]]

```
변경을 먼저 journal (log) 에 기록 → 디스크 적용 → log 삭제
crash 시 journal replay 로 일관성 회복
```

ext3/4, XFS, NTFS 등이 사용.

---

## 5. Copy-on-Write

자세히 → [[cow-snapshot]]

```
파일 변경 시 새 block 작성 → 메타데이터만 갱신
옛 block 은 snapshot 으로 보존 가능
```

Btrfs, ZFS, APFS, F2FS.

---

## 6. RAID

자세히 → [[raid]]

- RAID 0 — Stripe (속도, 안정성 0)
- RAID 1 — Mirror (안정성)
- RAID 5 — Parity (1 디스크 실패)
- RAID 6 — 2 Parity
- RAID 10 — Mirror + Stripe
- ZFS 의 RAIDZ — software-defined

---

## 7. fsync / Durability

자세히 → [[fsync-durability]]

```c
write(fd, ...);             // page cache 만
fsync(fd);                   // 디스크 동기화
fdatasync(fd);
sync_file_range(...);
sync();                      // 전체 시스템
```

**`write` 만으로는 디스크 미반영** — crash 시 손실.

---

## 8. VFS (Virtual File System)

Linux 의 추상 — `open/read/write` 가 어떤 FS 든 동일 인터페이스.

```
응용 →  VFS  →  ext4/XFS/NFS/FUSE/...
              file_operations 인터페이스
```

`/proc/filesystems` 로 등록된 FS 목록.

---

## 9. Page Cache

```
파일 read → 디스크 → kernel page cache (RAM)
같은 페이지 다시 read → cache hit
write → page cache 에 dirty mark → 백그라운드 flush
```

→ 대부분의 파일 I/O 는 RAM 의 page cache 사이.

`free -h` 의 `buff/cache` 가 이것.

---

## 10. 디렉토리 구조 (FHS)

```
/
├── bin    /sbin     /usr/bin  /usr/sbin    실행 파일
├── etc                                       설정
├── home                                      사용자
├── var    /log /lib /spool                  변경되는 데이터
├── tmp    /var/tmp                           임시
├── usr    /share /include                   읽기 전용 sys
├── opt                                       3rd-party
├── proc                                      가상 (커널)
├── sys                                       가상 (sysfs)
├── dev                                       디바이스
├── boot                                      커널 / initramfs
├── mnt    /media                             마운트
└── run                                       런타임 state
```

---

## 11. inotify / fanotify

파일 변경 감시:

```c
int fd = inotify_init();
inotify_add_watch(fd, "/path", IN_MODIFY | IN_CREATE | IN_DELETE);
// read fd → struct inotify_event
```

응용: 에디터의 reload, hot-reload, fswatch, ESET.

---

## 12. 세부 노트

| 노트 | 영역 |
| --- | --- |
| [[inode-dentry]] | inode / dentry / hardlink / symlink |
| [[journaling]] | journal modes / 복구 |
| [[cow-snapshot]] | COW / snapshot / rollback |
| [[ext-xfs-zfs]] | 주요 파일시스템 비교 |
| [[fsync-durability]] | fsync / O_DIRECT / barrier |
| [[raid]] | RAID levels |

---

## 13. 함정 (요약)

- `write` 만 신뢰 — fsync 필수
- 디렉토리도 fsync (생성된 파일 보존)
- ulimit -n / sysctl fs.file-max — FD limit
- noatime mount option (atime 갱신 비용)
- 작은 파일 폭증 → inode 고갈
- COW + huge file → fragmentation

---

## 14. 학습 자료

- **OSTEP** Ch. 39-45
- **The Linux Programming Interface** Ch. 13-19
- **Understanding the Linux Kernel** Ch. 12-18
- **Btrfs / ZFS** 공식 docs

---

## 15. 관련

- [[../io/io]] — I/O 모델
- [[../memory/mmap]] — page cache 통합
- [[../../database/postgresql/backup-recovery]] — fsync 와 DB durability
- [[../operating-system|↑ OS hub]]
