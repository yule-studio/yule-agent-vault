---
title: "RAID — Redundant Array of Independent Disks"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T13:00:00+09:00
tags:
  - operating-system
  - filesystem
  - raid
---

# RAID

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | RAID 0/1/5/6/10 + 운영 |

**[[filesystem|↑ Filesystem hub]]**

---

## 1. 한 줄

여러 디스크를 한 논리 디스크로 묶어 **속도 / 안정성** 을 얻음.

---

## 2. 주요 레벨

| Level | 최소 | 안정 | 속도 | 사용량 | 의미 |
| --- | --- | --- | --- | --- | --- |
| **0** | 2 | ❌ | ↑↑ | 100% | Stripe (병렬) |
| **1** | 2 | 1 디스크 OK | ↑ read | 50% | Mirror (복사) |
| **5** | 3 | 1 디스크 OK | ↑ read, parity 비용 | (N-1)/N | Stripe + 1 Parity |
| **6** | 4 | 2 디스크 OK | ↑ read | (N-2)/N | Stripe + 2 Parity |
| **10** | 4 | mirror 쌍 OK | ↑↑ | 50% | RAID 1+0 |

---

## 3. RAID 0 — Stripe

```
Block A → disk1
Block B → disk2
Block C → disk1
Block D → disk2
```

- 모든 disk 병렬 → 속도 N 배
- 1 디스크 실패 = 전체 손실

→ 캐시 / 임시 / 성능 우선.

---

## 4. RAID 1 — Mirror

```
disk1 = disk2 (완전 복사)
```

- read 는 분산 (속도 ↑)
- write 는 둘 다 → 같은 속도
- 1 디스크 실패 OK
- 사용량 50%

→ 작은 부팅 / 단순.

---

## 5. RAID 5 — Stripe + Parity

```
Block A1 A2 P(A)
Block B1 P(B) B3
Block P(C) C2 C3
... (parity 분산)
```

- 데이터 (N-1) + parity 1 분산
- 1 디스크 실패 OK (parity 로 복구)
- 사용량 (N-1)/N (75% with 4 disks)

### 5.1 Write Hole / RAID 5 Penalty

- Write 1 block = 새 parity 계산 → 2 read + 2 write
- crash 시 data + parity 불일치 가능 → "Write Hole"

→ RAID 5 의 write 가 느림. BBU + journal 권장.

### 5.2 큰 디스크에서 위험
디스크 1 실패 → rebuild 중 두 번째 실패 → 전체 손실. URE (Unrecoverable Read Error) 확률 ↑.

→ 큰 (> 4TB) HDD = RAID 6 권장.

---

## 6. RAID 6 — 2 Parity

```
Block A1 A2 P(A) Q(A)
```

- 2 디스크 동시 실패 OK
- 사용량 (N-2)/N
- write penalty 더 큼

→ 큰 array 의 표준.

---

## 7. RAID 10 — Mirror + Stripe

```
Group1: disk1 = disk2 (mirror)
Group2: disk3 = disk4 (mirror)
Stripe across groups
```

- 사용량 50%
- 쓰기 빠름 (parity 계산 X)
- 한 mirror 쌍 1 디스크씩은 OK
- DB / 운영 표준

비교: RAID 0+1 (먼저 stripe 후 mirror) — 약간 다름 + 더 위험.

---

## 8. 하드웨어 vs 소프트웨어 RAID

### 8.1 하드웨어
- 전용 카드 (LSI / Adaptec / HP Smart Array)
- BBU / NVRAM 으로 write cache 안전
- OS 무관
- 비싸고 vendor lock-in

### 8.2 소프트웨어 (Linux mdadm)
```bash
mdadm --create /dev/md0 --level=10 --raid-devices=4 /dev/sd[abcd]
cat /proc/mdstat
mdadm --detail /dev/md0
mdadm --add /dev/md0 /dev/sde
mdadm --fail /dev/md0 /dev/sda
mdadm --remove /dev/md0 /dev/sda
```

자유 / 유연 / 검증.

### 8.3 LVM RAID
LVM 의 RAID 통합. 같은 mdraid 기반.

### 8.4 ZFS RAIDZ / Btrfs RAID
파일시스템 안의 RAID — 더 똑똑 (체크섬으로 silent corruption 발견).

---

## 9. 핫 스페어

```bash
mdadm --add /dev/md0 /dev/sdz   # 자동 spare
mdadm --detail /dev/md0          # 'spare' 표시
```

디스크 실패 시 자동 rebuild 시작.

---

## 10. Scrub / Resilver

정기적으로 모든 disk 읽기 → silent corruption 검출.

```bash
echo check > /sys/block/md0/md/sync_action
echo idle > /sys/block/md0/md/sync_action     # 멈춤

# ZFS
zpool scrub tank
```

월 1 회 권장.

---

## 11. SSD + RAID

- SSD 의 wear leveling — RAID 1 시 같은 패턴 → 두 SSD 가 동시에 죽을 수 있음
- 다른 vendor / batch 권장
- TRIM 지원 RAID 권장
- write amplification 고려

---

## 12. RAID 의 한계

> "RAID is not backup"

RAID 가 보호 못 함:
- ransomware
- 인간 실수 (rm -rf)
- 컨트롤러 / 펌웨어 손상
- 전체 시스템 도난 / 화재

→ **별도 백업 + offsite + immutable** 필수.

---

## 13. 클라우드 (RAID 대안)

| | 전통 RAID | Cloud |
| --- | --- | --- |
| 안정성 | Disk redundancy | 데이터 복제 (EBS, S3, GCS) |
| 확장 | 수동 | API |
| Hot swap | 자동 | 자동 |
| 비용 | 디스크 + 컨트롤러 | 사용량 |

AWS EBS 는 내부적으로 multi-replica. RAID 가 응용 레벨에서 거의 불필요.

---

## 14. 분산 RAID — Erasure Coding

대규모 분산 시스템에선 단순 RAID 5/6 보다 일반화된 erasure coding:

```
k 데이터 + m parity → m 까지 손실 OK
```

- Reed-Solomon
- 사용량 k/(k+m) — 다양한 비율
- Ceph / GlusterFS / S3 backend / HDFS EC

---

## 15. 함정

### 15.1 RAID 0 운영
한 디스크 실패 = 전체. 백업 + 핵심 아닌 경우만.

### 15.2 큰 HDD RAID 5
URE 확률 + 긴 rebuild. RAID 6 / RAID 10.

### 15.3 SSD RAID 1 의 동시 사망
같은 wear → 같이 죽음. 다른 batch.

### 15.4 RAID 가 백업
ransomware / 실수에 무력.

### 15.5 nobarrier on RAID
캐시 cache 의 reorder + crash 일관성 X.

### 15.6 rebuild 중 두 번째 실패
RAID 5 → 전체 손실. hot spare + scrub.

### 15.7 fakeRAID (BIOS)
대부분 SW RAID 와 비슷한 성능. mdadm 이 더 안정.

### 15.8 ZFS + 하드웨어 RAID
ZFS 가 raw disk 가져야 — 컨트롤러는 JBOD / HBA.

---

## 16. 학습 자료

- **OSTEP** Ch. 38 (RAID)
- **mdadm man page**
- **Open ZFS** RAIDZ docs
- **The Pragmatic Programmer for Storage** — 시간 부족하면 ZFS / Btrfs / 클라우드 native

---

## 17. 관련

- [[ext-xfs-zfs]] — ZFS RAIDZ
- [[fsync-durability]]
- [[filesystem]] — FS hub
