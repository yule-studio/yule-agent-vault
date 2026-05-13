---
title: "Cassandra 데이터 모델링 — Partition / Clustering / Query-First"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T00:15:00+09:00
tags:
  - database
  - cassandra
  - modeling
---

# Cassandra 데이터 모델링

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | partition / clustering / 패턴 |

**[[cassandra|↑ Cassandra hub]]**

---

## 1. 한 줄

> "**Query-first**" — 쿼리에 맞춰 테이블 설계. RDB 의 정규화와 반대.

같은 데이터를 **여러 테이블에 denormalize** 하는 게 정상.

---

## 2. Primary Key 구조

```sql
PRIMARY KEY ((partition_key), clustering_key1, clustering_key2)
```

```
Row → 같은 partition_key 끼리 같은 노드
       partition 안에서 clustering_key 로 정렬
```

### 2.1 예

```sql
CREATE TABLE events (
  user_id    UUID,
  event_time TIMESTAMP,
  event_id   UUID,
  data       TEXT,
  PRIMARY KEY ((user_id), event_time, event_id)
) WITH CLUSTERING ORDER BY (event_time DESC, event_id ASC);
```

- partition: `user_id`
- 정렬: `event_time DESC` 후 `event_id ASC`

---

## 3. 좋은 Partition Key

| 속성 | 의미 |
| --- | --- |
| High cardinality | 분산 |
| 작은 / 균등 partition | hot partition X |
| 쿼리에 항상 포함 | targeted query |

### 3.1 나쁜 예

```sql
PRIMARY KEY ((country))   -- "KR" 이 90% → hot partition
PRIMARY KEY ((year))      -- 단조 증가 → 최신 노드만 hit
```

### 3.2 좋은 예

```sql
PRIMARY KEY ((user_id))
PRIMARY KEY ((tenant_id, bucket))    -- composite
```

---

## 4. Partition 크기 한계

- 권장: **< 100 MB / partition**
- < 100,000 cells / partition
- 한 partition 이 한 노드 디스크에 있음 → 너무 크면 노드 부담

### 4.1 Time Bucket — 시계열 패턴

```sql
CREATE TABLE events_by_user (
  user_id   UUID,
  bucket    TEXT,           -- '2026-05'
  event_time TIMESTAMP,
  data      TEXT,
  PRIMARY KEY ((user_id, bucket), event_time)
);

-- 쿼리
SELECT * FROM events_by_user
WHERE user_id = ? AND bucket = '2026-05'
  AND event_time > '2026-05-01';
```

→ 한 사용자의 한 달 데이터가 한 partition. 다음 달은 새 partition.

---

## 5. Clustering Key

partition 안의 정렬. 범위 쿼리 가능:

```sql
SELECT * FROM events
WHERE user_id = ?
  AND event_time > '2026-05-01'
  AND event_time < '2026-06-01';
```

- 첫 clustering 부터 prefix
- DESC / ASC 명시 — 쿼리 정렬에 사용

---

## 6. Query-First Modeling 단계

1. **모든 query 나열**
2. 각 query 마다 테이블 1 개
3. partition / clustering 결정
4. (denormalize OK) 같은 데이터 여러 테이블

### 6.1 예 — 메시지 앱

쿼리:
- Q1: 사용자가 받은 모든 메시지 (시간 순)
- Q2: 특정 대화의 모든 메시지

테이블:

```sql
-- Q1
CREATE TABLE messages_by_user (
  user_id      UUID,
  bucket       TEXT,
  message_time TIMESTAMP,
  message_id   UUID,
  from_user    UUID,
  body         TEXT,
  PRIMARY KEY ((user_id, bucket), message_time, message_id)
) WITH CLUSTERING ORDER BY (message_time DESC, message_id ASC);

-- Q2
CREATE TABLE messages_by_chat (
  chat_id      UUID,
  message_time TIMESTAMP,
  message_id   UUID,
  from_user    UUID,
  body         TEXT,
  PRIMARY KEY ((chat_id), message_time, message_id)
) WITH CLUSTERING ORDER BY (message_time DESC, message_id ASC);
```

→ 같은 메시지를 2 곳에 INSERT. **batch** 로 원자성.

---

## 7. Collection 타입

```sql
CREATE TABLE users (
  user_id UUID PRIMARY KEY,
  tags    SET<TEXT>,
  roles   LIST<TEXT>,
  attrs   MAP<TEXT, TEXT>,
  emails  FROZEN<SET<TEXT>>     -- 통째로 다룸
);

UPDATE users SET tags = tags + {'new'} WHERE user_id = ?;
UPDATE users SET emails = {'a@x.com','b@x.com'} WHERE user_id = ?;
```

⚠️ Collection 크기 작게 (수십~수백). 큰 collection → 별도 테이블.

---

## 8. UDT (User-Defined Type)

```sql
CREATE TYPE address (
  street TEXT,
  city   TEXT,
  zip    TEXT
);

CREATE TABLE users (
  user_id UUID PRIMARY KEY,
  home    FROZEN<address>
);
```

`FROZEN` = 통째로 (변경 시 전체 교체).

---

## 9. Counter 타입

```sql
CREATE TABLE page_views (
  page_id UUID PRIMARY KEY,
  views   COUNTER
);

UPDATE page_views SET views = views + 1 WHERE page_id = ?;
```

⚠️ Counter 만의 제약 — 다른 컬럼 / 일반 UPDATE 와 섞을 수 X.

---

## 10. Time-Based UUID (TIMEUUID)

```sql
CREATE TABLE events (
  bucket TEXT,
  id     TIMEUUID,
  data   TEXT,
  PRIMARY KEY ((bucket), id)
) WITH CLUSTERING ORDER BY (id DESC);

INSERT INTO events (bucket, id, data) VALUES ('2026-05', now(), '...');
SELECT * FROM events WHERE bucket = '2026-05'
  AND id < maxTimeuuid('2026-05-14');
```

시간 정렬 + 유일.

---

## 11. Secondary Index — 신중

```sql
CREATE INDEX idx_users_email ON users (email);
```

- 모든 노드에 인덱스 분산 — scatter-gather
- **low cardinality 에 부적합** (모든 노드 hit)
- 보통 별도 lookup 테이블이 더 좋음

### 11.1 대안 — Inverse Table

```sql
-- 추가 lookup
CREATE TABLE users_by_email (
  email   TEXT PRIMARY KEY,
  user_id UUID
);
```

INSERT 시 두 테이블에 batch.

### 11.2 SASI / SAI

- SASI = 옛 (deprecated)
- **SAI (Storage-Attached Index, 5.0+)** = 새 표준 — secondary 의 진화

---

## 12. Materialized View — 자동 denormalize

```sql
CREATE MATERIALIZED VIEW users_by_email AS
  SELECT * FROM users
  WHERE email IS NOT NULL AND user_id IS NOT NULL
  PRIMARY KEY (email, user_id);
```

⚠️ **시험적 / 결함** — 4.x 도 unstable. 수동 inverse table 권장.

---

## 13. LWT (Lightweight Transaction)

```sql
INSERT INTO users (user_id, email) VALUES (?, ?)
IF NOT EXISTS;

UPDATE users SET email = ? WHERE user_id = ?
IF email = ?;
```

Paxos 기반 — 4 round trip. **비싸다**. 꼭 필요할 때만 (예: 첫 등록 유일성).

---

## 14. BATCH

```sql
BEGIN BATCH
  INSERT INTO messages_by_user (...) VALUES (...);
  INSERT INTO messages_by_chat (...) VALUES (...);
APPLY BATCH;
```

### 14.1 종류

- **Logged Batch** (기본) — 원자성 (실패 시 모두 적용 후 retry)
- **Unlogged Batch** (`UNLOGGED`) — 성능, 원자성 X

⚠️ Cassandra BATCH 는 RDB BATCH 와 다름 — **multi-partition batch 는 비싸다**. 같은 partition 내에서만 권장.

---

## 15. 자주 쓰는 패턴

### 15.1 Time-Series

```sql
CREATE TABLE metrics (
  device_id UUID,
  bucket    TEXT,                 -- 'YYYY-MM-DD-HH'
  ts        TIMESTAMP,
  value     DOUBLE,
  PRIMARY KEY ((device_id, bucket), ts)
) WITH compaction = {
  'class':'TimeWindowCompactionStrategy',
  'compaction_window_size':1,
  'compaction_window_unit':'HOURS'
} AND default_time_to_live = 2592000;   -- 30d
```

### 15.2 Feed / Activity

```sql
CREATE TABLE feed (
  user_id    UUID,
  bucket     TEXT,
  posted_at  TIMESTAMP,
  post_id    UUID,
  data       TEXT,
  PRIMARY KEY ((user_id, bucket), posted_at, post_id)
);
```

### 15.3 Latest N

partition 안에서 가장 최신 N → ORDER BY clustering DESC + LIMIT.

```sql
SELECT * FROM feed WHERE user_id = ? AND bucket = ? LIMIT 20;
```

### 15.4 Multi-Tenant

```sql
PRIMARY KEY ((tenant_id, user_id), ...)
```

테넌트 격리 + partition 균등.

---

## 16. 안티패턴

### 16.1 RDB 처럼 정규화
JOIN X. denormalize.

### 16.2 Hot partition
같은 partition_key 폭증.

### 16.3 거대 partition
> 100 MB / 100K cells.

### 16.4 ALLOW FILTERING
운영 사용 X — 데이터 모델 잘못.

### 16.5 Secondary Index 남용
low cardinality. inverse table.

### 16.6 큰 collection
큰 list/set/map → 별도 테이블.

### 16.7 multi-partition BATCH
비쌈. 같은 partition 만.

### 16.8 LWT 남용
4 round trip. 정말 필요한 곳만.

---

## 17. 학습 자료

- **DataStax Academy** — Data Modeling course
- **Cassandra: The Definitive Guide** Ch. 4-6
- **Cassandra Data Modeling Best Practices** — DataStax blog
- **PlantUML / KDM** 데이터 모델링 도구

---

## 18. 관련

- [[cql-syntax]] — CQL
- [[consistency-tunable]] — CL
- [[cassandra]] — Cassandra hub
