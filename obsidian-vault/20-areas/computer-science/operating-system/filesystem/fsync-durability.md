---
title: "fsync / Durability — 디스크 동기화"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T12:55:00+09:00
tags:
  - operating-system
  - filesystem
  - fsync
  - durability
---

# fsync / Durability

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | fsync / fdatasync / O_DIRECT |

**[[filesystem|↑ Filesystem hub]]**

---

## 1. 한 줄

`write()` 는 **page cache 만** 갱신 — 디스크 아님. **`fsync(fd)` 가 진짜 디스크 동기화**.

crash 시 fsync 안 한 데이터 손실 가능.

---

## 2. write 의 흐름

```
write(fd, buf, n)
    → user buffer → kernel page cache (dirty)
    → return (즉시)

(나중) writeback thread (`pdflush` / `flush-*`):
    → dirty page → device
```

→ write 직후 응답 빠름. 그러나 디스크엔 아직 없음.

---

## 3. fsync / fdatasync

```c
fsync(fd);             // 데이터 + 메타 (inode size, mtime, ...)
fdatasync(fd);          // 데이터 + 필수 메타 만 (mtime 등 무시)
```

→ syscall 이 끝나면 device 에 영구화 (이론).

`fdatasync` 가 약간 빠름 (메타 일부 skip).

---

## 4. sync / syncfs

```c
sync();                 // 전체 시스템 — 모든 dirty page
syncfs(fd);             // fd 가 속한 fs 만
```

shutdown 직전 / 운영용.

---

## 5. 디렉토리도 fsync 필요

```
파일 생성 (rename, create, unlink) → 디렉토리의 dentry 변경
디렉토리 자체 fsync 안 하면 → 디렉토리 메타 손실 → 파일 보이지 않을 수 있음
```

```c
int fd = open("file.tmp", O_CREAT|O_WRONLY, 0644);
write(fd, data, n);
fsync(fd);
close(fd);

rename("file.tmp", "file");

// 부모 디렉토리도 fsync
int dirfd = open(".", O_RDONLY);
fsync(dirfd);
close(dirfd);
```

→ DB / 로그 / 안전한 atomic write 의 표준 패턴.

---

## 6. atomic file update

```c
// 임시 파일 → fsync → rename → 디렉토리 fsync
1. open("tmp", O_CREAT|O_WRONLY|O_TRUNC, 0644)
2. write(...)
3. fsync(tmpfd)
4. close(tmpfd)
5. rename("tmp", "target")
6. open(".", O_RDONLY); fsync(dirfd)
```

crash 시 — old 또는 new — 일관.

---

## 7. fsync 비용

- HDD: 10-50 ms (seek + rotation)
- SSD: 1-10 ms (flash erase)
- NVMe: 100-500 μs

DB 의 transaction commit = 1 fsync = 이 만큼 latency.

→ batching / group commit / async commit 으로 amortize.

---

## 8. O_DIRECT — page cache bypass

```c
int fd = open("data.bin", O_RDWR | O_DIRECT);
write(fd, buf, n);    // page cache 안 거치고 직접 디스크
```

- buf / size 가 block 정렬 (보통 512 / 4K)
- DB / 검색 엔진 사용 (자체 cache 관리)
- 일반 응용은 page cache 가 좋음

→ DB 가 OS cache 와 자체 cache 의 double caching 회피.

---

## 9. O_SYNC / O_DSYNC

```c
int fd = open("...", O_WRONLY | O_SYNC);
write(fd, ...);   // 매 write 가 fsync 까지
```

매번 fsync — 매우 느림. 거의 안 씀.

`O_DSYNC` = `fdatasync` 동등.

---

## 10. write barrier — 디스크 cache

```
디스크 (특히 HDD) 자체 cache → reorder write → crash 시 일관성 X
→ kernel 이 FLUSH 명령으로 barrier
```

`barrier=1` (기본). `nobarrier` 면 빠르지만 위험.

### 10.1 BBU / supercap
- BBU (Battery-Backed Unit) — RAID 카드
- Power-loss protection capacitor — enterprise SSD
- → fsync 가 더 빨라도 안전

consumer SSD 는 없을 수 있음 — fsync 후 손실 가능.

---

## 11. Linux 의 fsync 의 함정 (옛)

- ext3/4 `data=ordered` 기본 — fsync(파일) 이 fs 전체 flush 비슷한 부작용
- `data=writeback` 에선 fsync 가 더 가벼움
- ext4 의 일부 ext4 bug 가 과거 silent fsync 실패 → 데이터 손실 (Jepsen)

Modern Linux 는 거의 해결. 단, fsync 자체가 fail (ENOSPC, EIO) 도 가능 — **errno 확인 + 재시도 / abort**.

```c
if (fsync(fd) != 0) {
    // ⚠️ 데이터 손실 가능 — 보통 abort + 로그
    perror("fsync");
    abort();
}
```

PostgreSQL 의 "fsync gate" 사고 (2018) — fsync 실패 후에도 dirty page 가 cache 에 — 재시도 시 다른 에러.

---

## 12. eventfd / sync_file_range

```c
sync_file_range(fd, off, len, FLAGS);
```

특정 범위만 flush. Postgres 의 sync_file_range 사용.

- `SYNC_FILE_RANGE_WRITE` — 시작
- `SYNC_FILE_RANGE_WAIT_BEFORE/AFTER` — 대기

`fsync` 와 다름 — 메타 보장 X.

---

## 13. DB 의 durability 보장

### 13.1 PostgreSQL

```ini
fsync = on
synchronous_commit = on   # WAL fsync 까지
full_page_writes = on
wal_sync_method = fdatasync   # 또는 fsync / open_datasync
```

자세히 → [[../../database/postgresql/configuration#5-walcheckpoint]]

### 13.2 MySQL InnoDB

```ini
innodb_flush_log_at_trx_commit = 1   # 매 commit fsync
sync_binlog = 1
```

자세히 → [[../../database/mysql/innodb-engine]]

### 13.3 Redis

```ini
appendfsync everysec   # 1초 마다 fsync
```

자세히 → [[../../database/redis/persistence]]

---

## 14. Cloud / EBS / NFS

### 14.1 NFS
fsync 가 client → server fsync 까지 — 매우 느림.
NFS 의 일관성 모델 신뢰성 ↓ — DB 위험.

### 14.2 EBS / Cloud disk
보통 fsync 신뢰. 단, BBU 대신 cloud 의 replication.

### 14.3 ephemeral SSD
재부팅 후 사라짐 — durability 보장 X.

---

## 15. async I/O 에서

io_uring 의 `IORING_OP_FSYNC` — 비동기 fsync. 일반 응용은 충분.

자세히 → [[../io/io-uring]]

---

## 16. 함정

### 16.1 `write` 만 신뢰
crash 시 손실. fsync 필수.

### 16.2 디렉토리 fsync 누락
생성된 파일 사라질 수 있음.

### 16.3 fsync 의 비용 무시
batch commit / group commit.

### 16.4 fsync fail 무시
abort + 로그. PostgreSQL fsync gate.

### 16.5 `nobarrier` 사용
크래시 시 일관성 X.

### 16.6 consumer SSD 의 plp 가정
없을 수도 — enterprise SSD 권장 (DB).

### 16.7 NFS 위의 DB
잠금 / fsync 신뢰 X.

### 16.8 `O_DIRECT` 의 정렬
정렬 안 맞으면 EINVAL.

### 16.9 `O_SYNC` 남용
매 write fsync — 죽음.

---

## 17. 학습 자료

- **PostgreSQL fsync 문서** + 2018 fsync gate 글
- **LWN.net — fsync()** 시리즈
- **The Linux Programming Interface** Ch. 13
- **Jepsen — Postgres / MySQL** 리포트

---

## 18. 관련

- [[journaling]]
- [[../io/io-uring]]
- [[../../database/postgresql/configuration]]
- [[filesystem]] — FS hub
