---
title: "ext4 / XFS / Btrfs / ZFS / F2FS / APFS / NTFS 비교"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T12:50:00+09:00
tags:
  - operating-system
  - filesystem
---

# 파일 시스템 비교

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 주요 FS 비교 |

**[[filesystem|↑ Filesystem hub]]**

---

## 1. 한눈에

| FS | 출시 | OS | journal | COW | snapshot | 특징 |
| --- | --- | --- | --- | --- | --- | --- |
| **ext4** | 2008 | Linux | ✅ | ❌ | ❌ (LVM) | 표준 / 안정 |
| **XFS** | 1993 | Linux/IRIX | meta | ❌ | reflink | 큰 파일 / 고성능 |
| **Btrfs** | 2009 | Linux | COW | ✅ | ✅ | snapshot / RAID |
| **ZFS** | 2005 | Solaris/BSD/Linux | COW | ✅ | ✅ | 가장 풍부 |
| **F2FS** | 2012 | Linux | LSF | ✅ | ✅ | flash 친화 |
| **APFS** | 2017 | macOS | meta | ✅ | ✅ | macOS 기본 |
| **NTFS** | 1993 | Windows | $LogFile | ❌ | shadow copy | Windows 표준 |
| **FAT32 / exFAT** | 1996 / 2006 | 전부 | ❌ | ❌ | ❌ | USB / 호환 |

---

## 2. ext4

Linux 의 표준 디폴트.

```bash
mkfs.ext4 /dev/sda1
mount -o noatime,discard /dev/sda1 /mnt
```

### 2.1 특징
- Extent (4 의 핵심) — block 그룹의 연속 범위
- 64-bit (16 TB+ 파일)
- delayed allocation
- multi-block allocator
- htree (디렉토리 B-tree)
- journal (ordered / journal / writeback)

### 2.2 한계
- snapshot 없음 (LVM 으로)
- 큰 파일 시스템 (수 PB) 비효율
- COW 없음

### 2.3 운영
```bash
tune2fs -l /dev/sda1                     # 정보
e2fsck -f /dev/sda1                      # 검사
resize2fs /dev/sda1                       # 크기
```

---

## 3. XFS

큰 파일 / 큰 디스크 / 고성능. Red Hat / 클라우드 기본.

```bash
mkfs.xfs /dev/sda1
mount /dev/sda1 /mnt
```

### 3.1 특징
- 모든 게 B+ tree
- 64-bit 처음부터
- allocation group (병렬)
- 매우 큰 디렉토리 / 파일 효율
- reflink (4.16+)
- 메타 journal

### 3.2 한계
- 줄이기 (shrink) 어려움
- snapshot X (LVM)

### 3.3 운영
```bash
xfs_info /mnt
xfs_repair /dev/sda1
xfs_growfs /mnt
xfs_quota
```

---

## 4. Btrfs

COW + 풍부한 기능. Facebook / SUSE 에서 운영.

```bash
mkfs.btrfs /dev/sda /dev/sdb
mount /dev/sda /mnt
btrfs subvolume create /mnt/data
btrfs subvolume snapshot -r /mnt/data /mnt/data@v1
```

### 4.1 특징
- COW
- subvolume / snapshot
- RAID 0/1/10 (5/6 시험)
- 체크섬 (CRC32C)
- 압축 (zstd, lzo)
- send/receive
- online resize

### 4.2 한계
- RAID 5/6 not production
- 큰 파일 성능 X (CoW fragmentation)
- 일부 워크로드 (DB) 비추 — nodatacow mount

---

## 5. ZFS

가장 풍부 / 안정 / 검증. Sun → Oracle → OpenZFS.

```bash
zpool create tank mirror /dev/sda /dev/sdb
zfs create tank/data
zfs set compression=lz4 tank/data
zfs snapshot tank/data@v1
```

### 5.1 특징
- COW + ARC + ZIL
- pool (vdev) 추상화
- RAIDZ (1/2/3)
- 압축 (lz4, zstd)
- 암호화
- send/receive
- dedup (RAM 많이)
- 체크섬 (SHA-256)

### 5.2 한계
- 라이선스 (CDDL — Linux 와 호환 X) — DKMS 로 설치
- RAM 권장 1 GB / 1 TB
- 거대 — 단순 케이스엔 과대

### 5.3 운영
```bash
zpool status
zpool scrub tank
zfs list
zfs get all tank/data
```

---

## 6. F2FS

Samsung 의 Flash 친화 FS. Android / 임베디드.

```bash
mkfs.f2fs /dev/mmcblk0
```

특징:
- Log-structured (sequential write)
- multi-segment
- COW (제한)
- TRIM 통합

---

## 7. APFS

macOS 10.13+ 기본.

특징:
- COW + snapshot
- container / volume
- 암호화 (FileVault)
- nanosec timestamps
- crash protection
- Time Machine 지원

```bash
diskutil apfs list
diskutil apfs snapshotrestore /
tmutil snapshot
```

---

## 8. NTFS

Windows 표준.

특징:
- 메타 journal ($LogFile)
- 권한 (DACL)
- Volume Shadow Copy
- compression
- encryption (EFS)
- ADS (Alternate Data Stream)

Linux 의 ntfs-3g (read/write) / 최신 ntfs3 (커널).

---

## 9. tmpfs — RAM 위의 FS

```bash
mount -t tmpfs -o size=1G tmpfs /mnt
```

- RAM 위에 fs
- reboot 시 사라짐
- /tmp 자주 tmpfs
- shm_open 의 backing

---

## 10. procfs / sysfs / cgroupfs

가상 fs — 커널 정보 / 설정 인터페이스.

```bash
mount | grep -E 'proc|sys|cgroup'
# proc on /proc type proc
# sysfs on /sys type sysfs
# cgroup2 on /sys/fs/cgroup type cgroup2
```

- `/proc/$PID/...` — 프로세스 정보
- `/sys/...` — 디바이스 / 커널 설정
- `/sys/fs/cgroup` — cgroup v2

---

## 11. FUSE — 유저 공간 FS

응용이 fs 구현 (low performance but flexible).

```bash
# sshfs — SSH 위에 마운트
sshfs user@host:/path /mnt

# rclone mount
# encfs / gocryptfs
# s3fs
```

---

## 12. NFS / SMB

네트워크 FS.

```bash
# NFS (Linux/UNIX)
mount -t nfs server:/export /mnt
# /etc/exports

# SMB / CIFS (Windows / Samba)
mount -t cifs //server/share /mnt -o user=...
```

자세히 → [[../../network/network]]

---

## 13. 선택 가이드

| 시나리오 | 추천 |
| --- | --- |
| 일반 Linux 서버 | ext4 / XFS |
| 큰 DB / VM 이미지 | XFS |
| Snapshot 필요 | Btrfs / ZFS (또는 LVM) |
| 거대 스토리지 (수십 TB+) | ZFS / XFS |
| Flash / Android | F2FS |
| macOS | APFS |
| Windows | NTFS |
| USB / dual-boot | exFAT |
| 컨테이너 image | OverlayFS + ext4/xfs |
| 임시 / RAM | tmpfs |
| 네트워크 공유 | NFS (UNIX) / SMB (Windows) |

---

## 14. mount option 핵심

```bash
mount -o noatime,nodiratime           # access time 갱신 X
mount -o relatime                      # atime <= mtime 일 때만
mount -o discard                       # TRIM (SSD)
mount -o nodev,nosuid,noexec           # 보안 (외부 USB)
mount -o ro                            # 읽기 전용
mount -o sync                          # 모든 write 즉시 (느림, 안전)
mount -o async                         # 기본 — write 비동기
```

---

## 15. 함정

### 15.1 RAID 컨트롤러 + ZFS
ZFS 는 raw disk 가 효율. RAID 컨트롤러의 cache 가 ZFS COW 와 충돌.

### 15.2 ext4 의 errors=continue
crash 후 mount 가 read-only 로. `mount -o remount,rw`.

### 15.3 XFS shrink
불가. 데이터 백업 + 재포맷.

### 15.4 Btrfs RAID 5/6 production
unstable.

### 15.5 tmpfs 의 size limit
미설정 시 RAM 의 50% 까지. OOM 위험.

### 15.6 FUSE 의 성능
kernel fs 보다 훨씬 느림 (수 배).

### 15.7 NFS 의 lock
Cassandra / SQLite 등은 NFS 위에서 위험.

---

## 16. 학습 자료

- **OSTEP** Ch. 39-45
- **ZFS 공식 docs**
- **Btrfs wiki**
- **XFS docs** (Red Hat / SUSE)
- **APFS Reference** (Apple)

---

## 17. 관련

- [[cow-snapshot]]
- [[journaling]]
- [[fsync-durability]]
- [[filesystem]] — FS hub
