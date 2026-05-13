---
title: "SQLite 트랜잭션 / WAL / Journal Modes"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T23:40:00+09:00
tags:
  - database
  - sqlite
  - transaction
  - wal
---

# SQLite 트랜잭션 / WAL

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | ACID / WAL / 락 / 동시성 |

**[[sqlite|↑ SQLite hub]]**

---

## 1. ACID

| 속성 | 보장 |
| --- | --- |
| Atomicity | Rollback Journal 또는 WAL |
| Consistency | Constraints |
| Isolation | Serializable (writer 단일) |
| Durability | fsync (synchronous level) |

`PRAGMA synchronous = FULL` (기본) 이면 강한 durability. WAL + NORMAL 이 실무 표준.

---

## 2. BEGIN / COMMIT / ROLLBACK

```sql
BEGIN;                            -- DEFERRED 와 같음 — 즉시 락 X
  UPDATE ...
COMMIT;

BEGIN IMMEDIATE;                  -- 즉시 RESERVED lock
  ...
COMMIT;

BEGIN EXCLUSIVE;                  -- 즉시 EXCLUSIVE (옛 모드만 의미)
```

### 2.1 BEGIN vs BEGIN IMMEDIATE

| 종류 | 동작 |
| --- | --- |
| `BEGIN` (`DEFERRED`) | 처음 read 시 SHARED, 처음 write 시 RESERVED 시도 |
| `BEGIN IMMEDIATE` | 시작 시 RESERVED 즉시 — write 트랜잭션은 이것 사용 권장 |

⚠️ DEFERRED + 다중 write 가능 → upgrade 시 `SQLITE_BUSY` 위험. **write 의도면 IMMEDIATE**.

---

## 3. SAVEPOINT

```sql
BEGIN;
  INSERT INTO orders ...;
  SAVEPOINT s1;
  INSERT INTO items ...;
  ROLLBACK TO s1;
  RELEASE s1;
COMMIT;
```

중첩 가능.

---

## 4. Journal Mode

```sql
PRAGMA journal_mode;
PRAGMA journal_mode = WAL;
```

### 4.1 DELETE (기본)

```
Transaction:
  1. 변경 전 page → journal 파일에 백업
  2. DB 파일 변경
  3. COMMIT → journal 파일 삭제
  Crash 시 journal 로 rollback
```

- 동시 read + write X (writer 진행 중 reader 차단)

### 4.2 WAL (Write-Ahead Log) — 권장

```
Transaction:
  1. 변경을 -wal 파일에 append
  2. COMMIT → -wal 끝 마커
  3. Reader 는 DB + WAL 까지 본 시점 snapshot
  Checkpoint → WAL 의 변경을 DB 로 옮김
```

- **Reader 는 writer 막지 않음**
- **Writer 는 reader 막지 않음**
- 한 시점에 **writer 1** 뿐 (변함 없음)
- WAL 파일 + SHM (shared memory)

---

## 5. WAL 활성

```sql
PRAGMA journal_mode = WAL;
-- "wal"
```

영구화 — DB 파일 헤더에 기록. 다음 열 때도 WAL.

### 5.1 WAL 의 파일

```
app.db        -- 메인
app.db-wal    -- WAL
app.db-shm    -- shared memory
```

백업 / 이동 시 같이 (또는 checkpoint 후 분리).

### 5.2 Checkpoint

```sql
PRAGMA wal_checkpoint(TRUNCATE);
-- 또는
PRAGMA wal_autocheckpoint = 1000;       -- 1000 페이지마다 자동
```

| Mode | 의미 |
| --- | --- |
| `PASSIVE` (기본) | 읽기 / 쓰기 안 막음 |
| `FULL` | 모든 reader 끝날 때까지 |
| `RESTART` | + 다음 write 까지 |
| `TRUNCATE` | + WAL 파일 0 으로 잘림 |

---

## 6. 락 (DELETE journal mode)

| 락 | 의미 |
| --- | --- |
| `UNLOCKED` | 락 없음 |
| `SHARED` | read 중 |
| `RESERVED` | 쓰기 의도 (다른 SHARED OK) |
| `PENDING` | 쓰기 임박 (새 SHARED X) |
| `EXCLUSIVE` | 쓰는 중 |

### 6.1 WAL 모드의 락

- 다중 SHARED reader
- 1 writer
- writer 와 reader 가 서로 안 막음 (snapshot)

---

## 7. SQLITE_BUSY

다른 트랜잭션 lock 으로 wait:

```sql
PRAGMA busy_timeout = 5000;       -- 5초까지 wait, 그래도 안 되면 BUSY
```

### 7.1 처리 패턴

```python
for attempt in range(10):
    try:
        conn.execute("BEGIN IMMEDIATE")
        conn.execute("UPDATE ...")
        conn.execute("COMMIT")
        break
    except sqlite3.OperationalError as e:
        if "database is locked" in str(e):
            time.sleep(0.01 * 2 ** attempt)
            continue
        raise
```

---

## 8. Isolation Level

SQLite 는 항상 **Serializable** — phantom / dirty / non-repeatable 모두 막음.
대신 동시성은 단일 writer 한계.

### 8.1 read_uncommitted (shared cache 만)

```sql
PRAGMA read_uncommitted = 1;
```

같은 프로세스의 shared cache 안에서만 의미. 일반 환경 무관.

---

## 9. synchronous 조합

```sql
-- 안전 최우선
PRAGMA journal_mode = DELETE;
PRAGMA synchronous = FULL;

-- 권장 (WAL)
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;

-- 빠름 / 위험
PRAGMA journal_mode = MEMORY;
PRAGMA synchronous = OFF;
```

자세히 → [[configuration]]

---

## 10. 동시성 패턴

### 10.1 다중 reader + 1 writer
WAL 모드 + reader-writer-lock 자연스럽게.

### 10.2 다중 writer
SQLite 는 **한 번에 1 writer**. 큐 / 워커가 직렬화 또는 busy_timeout.

### 10.3 application-level write coalescing

```python
write_queue = Queue()
def writer():
    while True:
        items = []
        items.append(write_queue.get())
        while len(items) < 100:
            try: items.append(write_queue.get_nowait())
            except: break
        with conn:
            for item in items:
                conn.execute(...)
```

→ 1 트랜잭션에 100 write → 100배 throughput.

### 10.4 Litestream / LiteFS / Turso
SQLite 의 분산 / 복제 솔루션.

---

## 11. PRAGMA wal_checkpoint 정책

```sql
-- 운영 정기 — 큰 WAL 방지
PRAGMA wal_autocheckpoint = 1000;

-- 매뉴얼
SELECT * FROM sqlite_master WHERE 1=0;
PRAGMA wal_checkpoint(TRUNCATE);
```

WAL 이 너무 커지면:
- 메모리 사용 ↑
- reader 가 더 많은 페이지 읽음

---

## 12. 손상 복구

### 12.1 integrity_check

```sql
PRAGMA integrity_check;
PRAGMA quick_check;
```

### 12.2 .recover (CLI)

```bash
sqlite3 corrupted.db .recover > recover.sql
sqlite3 new.db < recover.sql
```

### 12.3 dump 후 재구성

```bash
sqlite3 corrupted.db .dump > dump.sql
sqlite3 new.db < dump.sql
```

---

## 13. 함정

### 13.1 BEGIN 후 write upgrade 실패
DEFERRED → write 시 SQLITE_BUSY. IMMEDIATE 사용.

### 13.2 WAL 파일 빠짐
백업 시 main 만 복사 → write 안 보임. checkpoint 후 또는 셋 다 복사.

### 13.3 long-running read
WAL 의 모든 reader 가 끝나야 checkpoint. WAL 폭증.

### 13.4 NFS / 네트워크 파일시스템
fcntl lock 비신뢰. **로컬 파일 시스템** 만.

### 13.5 멀티 프로세스 + WAL
같은 머신 OK. 다른 머신 X (SHM 공유 안 됨).

### 13.6 `synchronous = OFF`
crash 시 데이터 손상. 분석 / 임시만.

### 13.7 트랜잭션 안의 PRAGMA
일부 PRAGMA 는 트랜잭션 안에서 변경 X. 시작 시 설정.

---

## 14. 학습 자료

- **WAL Mode** — sqlite.org/wal.html
- **File Locking** — sqlite.org/lockingv3.html
- **ACID** — sqlite.org/transactional.html

---

## 15. 관련

- [[configuration]] — PRAGMA
- [[limitations]] — 동시성 한계
- [[sqlite]] — SQLite hub
