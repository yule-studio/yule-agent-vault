---
title: "Copy-on-Write / Snapshot / Rollback"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T12:45:00+09:00
tags:
  - operating-system
  - filesystem
  - cow
  - snapshot
---

# Copy-on-Write / Snapshot

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | COW / snapshot / clone |

**[[filesystem|↑ Filesystem hub]]**

---

## 1. 한 줄

파일 / block 수정 시 **원본을 보존하고 새 곳에 작성**, 메타데이터만 atomic 으로 갱신.
→ snapshot / rollback / replication 이 자연스러움.

---

## 2. 동작

```
파일 A 의 block X 수정:

[기존]
A → X (data)

[COW]
1. 새 block Y 에 변경된 데이터 작성
2. A → Y (atomic 메타 갱신)
3. X 는 옛 snapshot 에 매핑된 채 보존

→ crash 시 — old 또는 new — 둘 다 일관 (atomic).
```

---

## 3. 장점

- **Snapshot 즉시** — 메타 복사만 (O(1))
- **rollback 자유** — old snapshot 으로 메타 swap
- **Clone** — 같은 데이터를 가리키는 새 파일
- **journal 불필요** — atomic 메타 swap 으로 일관성
- **dedup / checksum** 자연스러움

## 4. 단점

- **fragmentation** — random write 가 매번 새 block → 자주 정리 필요
- **rewrite cost** — 1 byte 변경 = 4KB block rewrite
- **메타데이터 ↑** — block 참조 카운터 등

---

## 5. ZFS

```bash
# Pool 생성
zpool create tank /dev/sda /dev/sdb
zfs create tank/data

# 스냅샷
zfs snapshot tank/data@v1
zfs list -t snapshot

# 롤백
zfs rollback tank/data@v1

# 클론 (writable)
zfs clone tank/data@v1 tank/data-fork
zfs promote tank/data-fork

# 전송 (incremental backup)
zfs send tank/data@v1 | zfs recv backup/data
zfs send -i v1 tank/data@v2 | zfs recv ...
```

특징:
- ZIL (ZFS Intent Log) — sync write 가속
- ARC (Adaptive Replacement Cache) — RAM cache
- L2ARC — SSD cache
- RAIDZ — software RAID
- 압축 / 암호화 / dedup

---

## 6. Btrfs

```bash
mkfs.btrfs /dev/sda
mount /dev/sda /mnt
btrfs subvolume create /mnt/data

# 스냅샷
btrfs subvolume snapshot -r /mnt/data /mnt/data@v1

# 롤백 = subvol swap
mv /mnt/data /mnt/data.old
btrfs subvolume snapshot /mnt/data@v1 /mnt/data

# 클론
btrfs subvolume snapshot /mnt/data /mnt/data-clone

# send/receive
btrfs send /mnt/data@v1 | btrfs receive /backup/
```

특징:
- subvolume / snapshot
- 압축 (zstd)
- 체크섬
- RAID (1/10 안정, 5/6 시험적)
- send/receive

---

## 7. APFS (macOS)

- 2017 도입, macOS 10.13+
- COW + snapshot (Time Machine 의 기반)
- crypto 통합
- container / volume

```bash
# Time Machine snapshot
tmutil snapshot
tmutil listlocalsnapshots /
```

---

## 8. NILFS / F2FS

- **NILFS** — Log-structured + snapshot 자동
- **F2FS** — Flash 친화 log-structured

---

## 9. CoW 가 아닌 ext4 / XFS 의 대안

### 9.1 LVM snapshot

```bash
lvcreate -L 10G -n data vg0
lvcreate -L 1G -s -n data-snap vg0/data       # snapshot

# COW at block layer (LVM 이 처리)
```

성능 손해 (snapshot 활성 동안 모든 write 가 다른 위치).

### 9.2 dm-thin / device-mapper
컨테이너 storage 의 thin provisioning + snapshot.

---

## 10. Snapshot 활용

### 10.1 백업
```
즉시 snapshot → 백업 도구가 snapshot 에서 copy (느려도 OK)
```

### 10.2 시스템 업데이트
```
snapshot → 업데이트 → 문제 시 rollback
```

NixOS / openSUSE 의 Snapper 시스템.

### 10.3 DB 백업
```
DB freeze (flush) → fs snapshot → DB resume
→ snapshot 마운트하여 백업
```

매우 빠른 백업 윈도우.

---

## 11. Reflink — 파일 단위 COW clone

```bash
cp --reflink=always src dst
```

같은 데이터 block 을 공유 → 즉시. 수정 시 분리.

지원: Btrfs, XFS, ZFS, APFS, NTFS (2018+).

→ git clone 가속, image build 가속 (Docker overlayfs 비슷).

---

## 12. Docker / 컨테이너

OverlayFS / Btrfs / ZFS 의 image 레이어 = COW:

```
Base image (read-only)
+ container layer (writable)
write 시 upper layer 에 복사 후 수정
```

자세히 → [[../virtualization/container]]

---

## 13. 함정

### 13.1 Fragmentation
random write 가 잦으면 → 정기 defrag.

### 13.2 free space 계산 어려움
clone / snapshot 으로 같은 block 공유 — 실제 사용량 ≠ logical sum.

### 13.3 snapshot 너무 많음
참조 카운트 폭증 / metadata 부담.

### 13.4 BTRFS RAID 5/6
unstable — RAID 1/10 권장.

### 13.5 ZFS RAM 요구
ARC 가 RAM 많이 — 작은 시스템에선 부담.

### 13.6 fsync semantics
COW 에서는 fsync 가 다른 의미 — 메타 swap 시점 보장.

### 13.7 Hot snapshot 의 함정
snapshot 시 DB 가 mid-write — 같이 freeze 필요.

### 13.8 dedup 의 cost
ZFS dedup = RAM 매우 큼. 보통 끔.

---

## 14. 학습 자료

- **OSTEP** Ch. 43 (LFS) + 44
- **ZFS Best Practices Guide**
- **Btrfs wiki**
- **Open ZFS docs**

---

## 15. 관련

- [[journaling]] — COW 의 대안
- [[ext-xfs-zfs]]
- [[../virtualization/container]]
- [[filesystem]] — FS hub
