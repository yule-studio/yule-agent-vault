---
title: "LSM-Tree — Log-Structured Merge Tree"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:20:00+09:00
tags:
  - data-structure
  - lsm-tree
  - storage
---

# LSM-Tree

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Memtable / SSTable / Compaction |

**[[advanced|↑ Advanced]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

**Write 빠른** disk-based KV. Memtable (in-memory) + 여러 SSTable (sorted disk file). RocksDB / LevelDB / Cassandra / HBase.

---

## 2. 구조

```
Memtable (RAM, skip list / B-tree)
    ↓ flush (full)
SSTable Level 0 (immutable, on disk)
    ↓ compact
SSTable Level 1
    ↓ compact
SSTable Level 2
    ...
```

### 핵심
- **Write** — Memtable (메모리) + WAL (재시작 복구)
- Memtable 가득 시 — disk 로 flush (SSTable)
- Background compaction — SSTable 들 merge

---

## 3. SSTable — Sorted String Table

### 정의
- Disk file — 정렬된 key/value
- Immutable (불변)
- Index + Bloom filter 부속

### 구조
```
[block 1] [block 2] [block 3] ... [index block] [bloom filter] [footer]
```

### 효과
- Sorted — binary search
- Bloom filter — 빠른 "없음" 체크
- Block — compression / cache

---

## 4. Write 흐름

```
1. Client write (key, value)
2. WAL append (sequential disk write)
3. Memtable insert (in-memory)
4. ACK to client
5. Memtable full → flush to disk (SSTable Level 0)
6. WAL delete (해당 부분)
```

### 효과
- Sequential disk write — 매우 빠름
- Random update X (모두 append)

---

## 5. Read 흐름

```
1. Memtable lookup
2. Memtable 에 없음 → Level 0 SSTables (모두, 시간 순)
3. Level 1 SSTable
4. Level 2 SSTable
...

각 SSTable:
  Bloom filter — 없음? skip
  Index — block 위치
  Block — binary search
```

### 효과
- Worst — 모든 level 검색
- Bloom — 대부분 skip
- 그래도 — B-tree 보다 느림

### Read amplification
- 한 key — 여러 level 검색
- 큰 LSM — 5-10× 더 느림 (vs B-tree)

---

## 6. Compaction — 통합

### 목적
- 옛 / 중복 key 정리
- Level 깊이 ↓
- Read 효율 ↑

### Size-Tiered (Cassandra default)
- 비슷한 크기 — 합치기
- Big files → 더 큰 file

### Leveled (RocksDB default)
- Level k — Level k+1 의 1/10 크기
- 한 SSTable 의 key range 다른 SSTable 와 겹치지 X
- Write amplification ↑, Read latency ↓

### Time-windowed (TWCS)
- Time-series 친화
- 시간 window 별 별도

### 함정 — Write amplification
- 한 key — 여러 번 rewrite (compaction)
- 큰 amplification — 10-30× (Leveled)

---

## 7. Bloom Filter — SSTable

### 목적
- "이 key 가 이 SSTable 에 있나?" 빠른 체크
- 없으면 — 다음 SSTable

### 효과
- 99%+ skip (FP 1% 정도)
- Read latency ↓ 매우 큼

자세히 → [[bloom-filter]]

---

## 8. WAL — Write-Ahead Log

### 정의
- Disk 의 sequential log
- Memtable insert 전에 — log 먼저

### 복구
- Crash 시 — Memtable 잃음
- WAL replay → Memtable 재구축

### 효과
- Durability + 빠른 write

---

## 9. LSM vs B-Tree

| | LSM-Tree | B-Tree |
| --- | --- | --- |
| **Write** | 매우 빠름 (sequential) | 느림 (random) |
| **Read** | 느림 (여러 level) | 빠름 |
| **Space amp** | 큼 (compaction 전) | 작음 |
| **Write amp** | 매우 큼 (compaction) | 보통 |
| **Compaction** | Background 필요 | 없음 |
| **사용** | Write-heavy | Read-heavy |

### 선택
- Time-series / logging — LSM
- OLTP RDBMS — B-Tree
- 모던 NoSQL — LSM 우세

---

## 10. 사용 — DB

### RocksDB (Facebook)
- LevelDB fork
- 매우 다양한 옵션
- Cassandra / MySQL MyRocks / etcd

### LevelDB (Google)
- Original, 단순

### Cassandra
- LSM + ring
- 분산

### HBase
- LSM (BigTable-inspired)

### ScyllaDB
- C++ Cassandra reimplementation

### SQLite — B-Tree (아님)
- 작은 / single-file

---

## 11. Optimization

### Compression
- Block 별 — Snappy / LZ4 / Zstd

### Block cache
- Hot SSTable block — RAM
- LRU

### Bloom filter
- 모든 SSTable — Bloom

### Hash sharding
- 큰 key 공간 — shard
- 각 shard — 작은 LSM

---

## 12. Tombstone

### 정의
- Delete 시 — "delete marker" insert (즉시 삭제 X)
- Compaction 시 — 옛 + tombstone 같이 제거

### 함정
- Many deletes — tombstone 누적
- Range scan — tombstone 도 봐야 → 느려짐
- TTL / 짧은 lifecycle 정책

---

## 13. Time-Series 특화

### TWCS (Time Window Compaction Strategy)
- 시간 window 별 — 별도 SSTable
- 옛 데이터 — 변경 안 됨

### 사용
- InfluxDB / Cassandra / ScyllaDB

---

## 14. Block Bloom / 변형

### Partitioned Index
- 큰 index — 부분 로딩

### Block Bloom
- Cache line 안 — Bloom
- Memory bandwidth ↓

---

## 15. Snapshot / Backup

### LSM 친화
- SSTable immutable — copy on snapshot
- 빠른 backup

### B-Tree
- WAL-based snapshot 필요

---

## 16. 함정

### 함정 1 — Write amplification
한 write — 10-30 disk write (compaction). SSD 수명 ↓.

### 함정 2 — Read tail latency
Memtable + L0 + L1 + ... — 큰 p99.

### 함정 3 — Compaction stall
Disk I/O 가득 — 더 이상 flush X. 큰 latency spike.

### 함정 4 — Bloom filter false positive
1% FP — 가끔 disk read.

### 함정 5 — Range query 의 비싸함
모든 level 모두 봐야. Index 가 도움.

### 함정 6 — SSD 의 wear
Compaction → 많은 write. Endurance 고려.

---

## 17. 학습 자료

- "The Log-Structured Merge-Tree (LSM-Tree)" — Patrick O'Neil 1996
- "BigTable" — Google paper 2006
- RocksDB wiki / docs
- DDIA Ch 3

---

## 18. 관련

- [[advanced]] — Hub
- [[../trees/b-tree]] — 비교
- [[bloom-filter]] — 핵심 부속
- [[skip-list]] — Memtable 구현
