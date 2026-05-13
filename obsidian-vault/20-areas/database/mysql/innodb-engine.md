---
title: "MySQL InnoDB 엔진 — Buffer Pool / Redo Log / Undo / Clustered Index"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T19:35:00+09:00
tags:
  - database
  - mysql
  - innodb
---

# MySQL InnoDB 엔진

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | InnoDB 구조 + Buffer Pool + Log |

**[[mysql|↑ MySQL hub]]**

---

## 1. 한 줄

InnoDB = **MySQL 의 표준 스토리지 엔진**. ACID, MVCC, 행 락, FK, crash recovery.
MyISAM 은 더 이상 운영에 쓰지 않는다 — InnoDB 가 사실상 유일한 선택.

---

## 2. 스토리지 엔진 비교

| 엔진 | 트랜잭션 | 락 | FK | crash 복구 | 용도 |
| --- | --- | --- | --- | --- | --- |
| **InnoDB** | ✅ | 행 | ✅ | ✅ | 모든 운영 |
| MyISAM | ❌ | 테이블 | ❌ | ❌ | 레거시만 |
| Memory | ❌ | 테이블 | ❌ | ❌ | 임시 |
| Archive | ❌ | 행 | ❌ | ✅ | 로그 |

```sql
SHOW ENGINES;
ALTER TABLE old_table ENGINE = InnoDB;
```

---

## 3. InnoDB 아키텍처

```
                       ┌────────────────────────────┐
        Client →  SQL Layer (parser, optimizer)
                       │
                       ▼
        ┌─────────────────────────────┐
        │   InnoDB Engine             │
        │ ┌──────────────────────┐    │
        │ │   Buffer Pool        │    │ ← 메인 캐시 (RAM)
        │ │ - data pages         │    │
        │ │ - index pages        │    │
        │ │ - change buffer      │    │
        │ │ - adaptive hash      │    │
        │ └──────────────────────┘    │
        │ ┌──────────────────────┐    │
        │ │   Redo Log           │    │ ← WAL (durability)
        │ │   Undo Log           │    │ ← MVCC rollback
        │ │   Binlog (server)    │    │ ← replication
        │ └──────────────────────┘    │
        └──────────────┬──────────────┘
                       ▼
        ┌─────────────────────────────┐
        │   Disk (ibd / redo / undo)  │
        └─────────────────────────────┘
```

---

## 4. Buffer Pool

데이터 + 인덱스의 **메인 메모리 캐시**.

### 4.1 크기

```ini
innodb_buffer_pool_size = 12G       # RAM 의 50-70%
innodb_buffer_pool_instances = 8    # 큰 buffer pool 의 lock 분산
```

### 4.2 LRU 변형

- **Young** (앞 5/8) — 자주 쓰는 페이지
- **Old** (뒤 3/8) — 갓 들어온 페이지
- 들어온 페이지가 일정 시간 (`innodb_old_blocks_time = 1000ms`) 안에 다시 쓰이면 young 으로 승격
- → **테이블 스캔이 hot 페이지를 밀어내지 않게** 보호

### 4.3 모니터링

```sql
SHOW ENGINE INNODB STATUS\G

SHOW STATUS LIKE 'Innodb_buffer_pool_%';

-- 적중률
SELECT 
  ROUND((1 - (s.read / r.reads)) * 100, 2) AS hit_pct
FROM
  (SELECT VARIABLE_VALUE AS read  FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') s,
  (SELECT VARIABLE_VALUE AS reads FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests') r;
-- > 99% 권장
```

---

## 5. Redo Log (WAL)

쓰기 보장 (Durability) — 변경을 먼저 redo log 에 기록 후 buffer pool 갱신.

```ini
innodb_redo_log_capacity = 4G          # 8.0.30+
innodb_log_file_size = 1G              # 8.0.30 이전 (file 당)
innodb_log_files_in_group = 2
innodb_flush_log_at_trx_commit = 1     # 1 = ACID
```

### 5.1 동작

```
COMMIT 시:
  1. redo log buffer → redo log file (write)
  2. fsync (flush_log_at_trx_commit=1 시)
  3. 클라이언트에 OK

이후 (별도):
  4. buffer pool 의 dirty page → 디스크 (lazy, checkpoint)
```

→ 디스크에 데이터 안 써도 redo 만 있으면 복구 가능.

### 5.2 innodb_flush_log_at_trx_commit

| 값 | 동작 |
| --- | --- |
| `1` (기본) | 매 commit fsync — 표준 ACID |
| `2` | 매 commit write, 1 초마다 fsync (OS crash 시 1 초 손실) |
| `0` | 1 초마다 write+fsync (MySQL crash 시 1 초 손실) |

운영 = 1. 분석 / 임시 = 2 검토.

---

## 6. Undo Log

이전 버전 보관 → MVCC + ROLLBACK.

```ini
innodb_undo_tablespaces = 2
innodb_undo_log_truncate = ON
innodb_max_undo_log_size = 1G
```

### 6.1 동작

```
UPDATE user SET name='B' WHERE id=1;

내부:
  1. 기존 row 를 undo log 에 복사
  2. row 를 새 값으로 갱신 (clustered index 안에서)
  3. transaction ID + undo pointer 기록

다른 트랜잭션이 읽으면:
  - 자기 시점보다 새 row → undo log 따라가 옛 값 복원
```

### 6.2 Long Transaction 의 위험

오래 열린 트랜잭션 → undo log 가 못 정리됨 → 디스크 ↑, 다른 read 도 느려짐.

```sql
SHOW ENGINE INNODB STATUS\G
-- "History list length" 가 크면 undo 누적
```

---

## 7. Change Buffer (Insert Buffer)

비-유니크 secondary 인덱스의 **변경을 모았다가 한꺼번에** 디스크 적용.

```ini
innodb_change_buffering = all     # all / inserts / deletes / changes / purges / none
innodb_change_buffer_max_size = 25  # buffer pool 의 25%
```

작은 무작위 쓰기를 일괄로 → 디스크 시킹 ↓. SSD 만 쓰면 효과 작음.

---

## 8. Adaptive Hash Index

자주 접근하는 인덱스 페이지 위에 자동 hash 인덱스 생성. 자동.

```ini
innodb_adaptive_hash_index = ON
```

워크로드 따라 off 가 더 빠를 수도 (Percona 권장).

---

## 9. 클러스터드 인덱스 (Clustered Index)

InnoDB 의 핵심 — **PK 가 곧 데이터 저장 순서**.

```
B+ Tree 리프 노드:
  PK 값 + 모든 컬럼 데이터
```

→ PK 검색 = 1 번 트리 traversal.
→ Secondary 인덱스의 리프는 **PK 값** 만 — secondary 검색 후 다시 PK 트리.

### 9.1 PK 선택의 함의

- **작고 단조 증가** 가 좋음 (BIGINT AUTO_INCREMENT, UUIDv7)
- 큰 PK → secondary 인덱스도 큼
- 무작위 PK (UUIDv4) → page fragmentation

### 9.2 PK 없으면?
InnoDB 가 숨겨진 6-byte ROW_ID 사용. 명시 PK 가 항상 좋음.

---

## 10. 페이지 / Extent / 세그먼트

```
Page (16 KB)         — 최소 I/O 단위
Extent (1 MB)        — 64 pages
Segment              — leaf segment + non-leaf segment 등
Tablespace (.ibd)    — 테이블당 또는 system
```

### 10.1 innodb_file_per_table

```ini
innodb_file_per_table = ON     # 기본
```

테이블당 .ibd 파일 → 운영 / 백업 / TRUNCATE 자유.

---

## 11. 락

### 11.1 락 종류

| 락 | 의미 |
| --- | --- |
| **Record Lock** | 인덱스 레코드 자체 |
| **Gap Lock** | 인덱스 사이 빈 공간 (Phantom 방지) |
| **Next-Key Lock** | Record + Gap (Repeatable Read 의 기본) |
| **Intention Lock** | 테이블 수준 의도 표시 |
| **AUTO-INC Lock** | AUTO_INCREMENT 직렬화 |

### 11.2 Phantom Read 방지 — Next-Key Lock

REPEATABLE READ 에서 `SELECT ... FOR UPDATE` 가 범위 락:

```sql
-- 다른 트랜잭션이 그 범위에 INSERT 불가
SELECT * FROM users WHERE age BETWEEN 20 AND 30 FOR UPDATE;
```

자세히 → [[transactions]]

---

## 12. Checkpoint

dirty page 를 디스크에 쓰는 작업.

- **Sharp Checkpoint** — redo log 끝나면 한꺼번에
- **Fuzzy Checkpoint** (기본) — 점진적 (LRU + dirty page list)

```ini
innodb_io_capacity = 2000           # SSD 권장
innodb_io_capacity_max = 4000
innodb_flush_neighbors = 0          # SSD 면 0 (HDD 1)
innodb_max_dirty_pages_pct = 75
```

---

## 13. Crash Recovery

```
시작:
  1. redo log 읽기 → committed 변경 재실행 (forward)
  2. undo log 읽기 → uncommitted 롤백 (backward)
  3. binlog 와 일관성 확인 (XA recovery)
```

`innodb_log_files / innodb_redo_log_capacity` 가 크면 복구 시간 ↑ 하지만 안 잃음.

---

## 14. 압축 (Compression)

```sql
CREATE TABLE big (...) ROW_FORMAT=COMPRESSED KEY_BLOCK_SIZE=8;
```

옵션:
- `ROW_FORMAT=COMPACT` (기본)
- `DYNAMIC` (8.0 기본) — TEXT/BLOB off-page
- `COMPRESSED` — zlib 압축

CPU vs 디스크. 콜드 데이터에 좋음.

---

## 15. 파티션

```sql
CREATE TABLE logs (
    id BIGINT, created_at DATETIME, ...
) PARTITION BY RANGE (YEAR(created_at)) (
  PARTITION p2024 VALUES LESS THAN (2025),
  PARTITION p2025 VALUES LESS THAN (2026),
  PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

| 종류 | 의미 |
| --- | --- |
| RANGE | 범위 (날짜 등) |
| LIST | 리스트 |
| HASH | 해시 |
| KEY | 해시 (자동 함수) |

### 15.1 한계
- FK 불가
- partition pruning 안 되면 의미 없음
- 8.0 부터 InnoDB 가 partition 직접 처리

---

## 16. 함정

### 함정 1 — `MyISAM` 사용
트랜잭션 X, crash 시 데이터 손상. InnoDB 만.

### 함정 2 — PK 무작위 (UUIDv4)
페이지 fragmentation, 인덱스 비대. BIGINT AUTO_INCREMENT 또는 UUIDv7 / ULID.

### 함정 3 — Buffer pool 너무 작음
기본 128 MB. 운영에 부족.

### 함정 4 — Long transaction → undo 누적
`SHOW ENGINE INNODB STATUS` 의 history list length 감시.

### 함정 5 — `innodb_flush_log_at_trx_commit=0/2` 의 위험
crash 시 데이터 손실. 의도적 선택만.

### 함정 6 — `innodb_file_per_table=OFF` 의 단일 ibdata
TRUNCATE / DROP 후에도 줄지 않음. 항상 ON.

### 함정 7 — Adaptive Hash 의 경합
일부 워크로드에서 lock 경합. Percona 는 off 권장하기도.

---

## 17. 학습 자료

- **High Performance MySQL** Ch. 4 (InnoDB Engine)
- **MySQL Reference Manual** Ch. 17 (InnoDB)
- **Percona Blog** — InnoDB 내부
- **MySQL Internals Manual**

---

## 18. 관련

- [[indexes]] — clustered / secondary
- [[transactions]] — 락 / MVCC
- [[performance-tuning]] — Buffer Pool 튜닝
- [[mysql]] — MySQL hub
