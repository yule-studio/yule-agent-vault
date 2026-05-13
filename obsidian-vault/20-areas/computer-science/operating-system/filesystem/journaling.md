---
title: "Journaling — 일관성 / 복구"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T12:40:00+09:00
tags:
  - operating-system
  - filesystem
  - journaling
---

# Journaling

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | journal modes / 복구 |

**[[filesystem|↑ Filesystem hub]]**

---

## 1. 한 줄

파일 변경을 **먼저 log (journal) 에 기록** 후 본 위치에 적용. crash 시 journal replay 로 일관성 회복.

---

## 2. 왜

```
파일 write 는 여러 메타데이터 변경 (inode, block bitmap, data block, dir entry)
→ 중간에 crash 시 일부만 반영 → 일관성 깨짐 → fsck (오래 걸림)
```

journal 로 atomic 단위 보장.

---

## 3. 동작

```
1. journal 에 변경 사항 기록 (log)
2. journal commit (마커 기록 + flush)
3. 본 위치에 적용 (checkpoint)
4. journal 의 해당 entry 비움

crash 시:
- 1 후 2 전 → 무시 (적용 X)
- 2 후 3 전 → replay (재적용)
- 3 후 4 전 → 다시 replay 해도 OK (idempotent)
```

→ Write-Ahead Log 와 같은 원리. DB 의 WAL 도 동일.

---

## 4. Journal Mode (ext3/4)

```bash
mount -o data=journal /dev/sda1 /mnt        # 모든 데이터 + 메타 journal
mount -o data=ordered /dev/sda1 /mnt        # 기본 — 메타 journal, 데이터 먼저 본 위치
mount -o data=writeback /dev/sda1 /mnt      # 메타만 journal — 빠름, 데이터 inconsistency 가능
```

| Mode | journal 단위 | 안전 | 성능 |
| --- | --- | --- | --- |
| `journal` | 메타 + 데이터 | ★★★ | ★ |
| `ordered` (기본) | 메타 | ★★ | ★★ |
| `writeback` | 메타만 — 순서 X | ★ | ★★★ |

→ 대부분 `ordered`. 데이터 안전 강해도 되면 `journal` (DB 옵션).

---

## 5. XFS / Btrfs / ZFS

- **XFS** — 메타데이터 journaling (writeback 비슷, 빠름)
- **Btrfs / ZFS** — Copy-on-Write — journal 불필요 (전체가 atomic update)

자세히 → [[cow-snapshot]]

---

## 6. crash 후 복구

```bash
# 부팅 시 자동
fsck -y /dev/sda1
mount /dev/sda1 /mnt
# journal replay (자동)
```

XFS 의 `xfs_repair`, Btrfs 의 `btrfs check`.

---

## 7. journal 위치

```bash
# ext4 — 같은 디바이스에 reserve
mke2fs -t ext4 -J size=128 /dev/sda1

# 또는 외부 디바이스 (성능 ↑)
mke2fs -O journal_dev /dev/sdb1
mke2fs -t ext4 -J device=/dev/sdb1 /dev/sda1
```

→ NVMe + HDD 조합에 effective.

---

## 8. Filesystem 별 journal

| FS | journal |
| --- | --- |
| ext3/4 | jbd / jbd2 (Journaling Block Device) |
| XFS | XFS Journal — 메타데이터 |
| NTFS | $LogFile |
| HFS+ | journal partition |
| ReiserFS | metadata journal |
| Btrfs / ZFS | COW 로 대체 |
| F2FS | NAT / SIT log |

---

## 9. fsync 와 journal

```c
fsync(fd);
```

- 본 위치 flush + journal flush
- journaling fs 에선 journal 의 해당 transaction commit 까지

→ fsync 가 매우 비쌈 (수 ms).

자세히 → [[fsync-durability]]

---

## 10. 동기 vs 비동기 commit

```bash
# ext4
mount -o commit=5 /dev/sda1 /mnt    # 5초마다 (기본 5)
```

`fsync` 가 안 불리면 5초 안의 데이터 손실 가능 — 일반 데스크탑에선 OK.

---

## 11. journal 크기

```bash
tune2fs -l /dev/sda1 | grep -i journal
# Journal size:    128M
```

작으면 자주 fill → write stall.
크면 RAM cache 활용 ↑.

128 MB ~ 1 GB 정도가 보통.

---

## 12. write barrier

```
journal commit 후 본 위치 적용 사이에 디스크 cache 가 reorder 하면 일관성 X
→ barrier 명령으로 순서 강제
```

NVMe / 모던 디스크는 barrier 보장. 단, 일부 설정 (`nobarrier`) 면 빠르지만 위험.

→ DB 워크로드는 **반드시 barrier on** (`barrier=1`).

---

## 13. Log-structured FS

journaling 의 극단 — **모든 write 가 log** (in-place 안 함). cleaner / GC 가 정리.

- F2FS (Flash)
- NILFS (Linux)
- Sprite LFS (1991, 학술)
- ZFS / Btrfs (COW + log 비슷)

→ random write → sequential write 변환. SSD / Flash 친화.

---

## 14. WAL (Write-Ahead Log) — DB 와의 관계

DB 의 WAL = journal 과 같은 개념:

| | FS journal | DB WAL |
| --- | --- | --- |
| 단위 | block / inode | row / page |
| 위치 | 디스크 reserve | 별도 파일 (pg_wal, mysql-bin) |
| 복구 | replay | replay |

→ FS journal 위에 DB WAL → **두 번 fsync** → 비용 ↑.
→ DB 가 raw device 사용 / `O_DIRECT` 로 FS cache 회피.

자세히 → [[../../database/postgresql/configuration#5-wal--checkpoint]]

---

## 15. 함정

### 15.1 `nobarrier` 위험
디스크 cache 가 fsync 거짓말 → crash 시 일관성 X.

### 15.2 `data=writeback` 사용 시
crash 후 메타는 OK 지만 데이터는 garbage 가능.

### 15.3 RAID 컨트롤러의 cache
battery-backed (BBU) 없으면 fsync 신뢰 X.

### 15.4 SSD 의 powerloss capacitor
consumer SSD 는 없을 수 있음 — fsync 후에도 손실 가능.

### 15.5 ZFS / Btrfs 의 journal 환상
COW — journal 다른 의미. 다른 복구 도구.

### 15.6 journal 크기 너무 작음
write spike 시 stall.

### 15.7 commit 시간 길게
fsync 안 부르는 워크로드의 손실 윈도우 ↑.

---

## 16. 학습 자료

- **OSTEP** Ch. 42 (Journaling)
- **Documentation/filesystems/ext4.rst**
- **XFS** internals
- **Soft Updates vs Journaling** (FreeBSD)

---

## 17. 관련

- [[fsync-durability]]
- [[cow-snapshot]]
- [[../../database/postgresql/configuration]]
- [[filesystem]] — FS hub
