---
title: "PostgreSQL 복제 — Streaming / Logical / Synchronous"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T18:40:00+09:00
tags:
  - database
  - postgresql
  - replication
  - ha
---

# PostgreSQL 복제

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Streaming / Logical / Sync / Hot Standby / Failover |

**[[postgresql|↑ PostgreSQL hub]]**

---

## 1. 복제 종류

| 종류 | 단위 | 용도 |
| --- | --- | --- |
| **Streaming (Physical)** | WAL byte 단위 | HA / 읽기 분산 / DR |
| **Logical** | row 변경 단위 | 부분 복제 / 버전 업그레이드 / CDC |
| **Trigger 기반** | 응용 레벨 | 레거시 (Slony, Bucardo) |

---

## 2. Streaming Replication (Physical)

### 2.1 구조

```
Primary (write)
  ↓ WAL stream
Standby 1 (read-only)
Standby 2 (read-only)
```

- WAL 그대로 송신 → standby 가 적용 (replay)
- 두 인스턴스는 **바이너리 동일**
- 같은 major version 필요

### 2.2 Primary 설정

```ini
# postgresql.conf
wal_level = replica           # 또는 logical (logical 이 streaming 도 포함)
max_wal_senders = 10
wal_keep_size = 1GB           # 또는 replication slot 사용
hot_standby = on
```

```
# pg_hba.conf
host replication repl_user 10.0.0.0/16 scram-sha-256
```

```sql
CREATE ROLE repl_user WITH REPLICATION LOGIN PASSWORD '...';
```

### 2.3 Standby 초기화

```bash
pg_basebackup \
  -h primary-host \
  -U repl_user \
  -D /var/lib/postgresql/16/main \
  -X stream \
  -R \                  # 자동으로 standby.signal + primary_conninfo 작성
  -P
```

### 2.4 Replication Slot

```sql
-- Primary 에서
SELECT pg_create_physical_replication_slot('standby_1');

-- Standby 의 postgresql.auto.conf
primary_slot_name = 'standby_1'
```

- WAL 보존 보장 — Standby 가 늦어도 안전
- 단점: Standby 가 죽으면 Primary 의 WAL 무한 증가 → 모니터링 필수

### 2.5 상태 확인

```sql
-- Primary
SELECT pid, usename, application_name, state, sync_state,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes
FROM pg_stat_replication;

-- Standby
SELECT pg_is_in_recovery();
SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;
```

---

## 3. Synchronous vs Asynchronous

### 3.1 비동기 (기본)

```
Primary 가 COMMIT 즉시 클라이언트에 응답
→ Standby 가 받기 전에 죽으면 마지막 트랜잭션 손실 가능
```

### 3.2 동기

```ini
synchronous_standby_names = 'ANY 1 (standby_1, standby_2)'
synchronous_commit = on
```

- 최소 1 개의 standby 가 WAL flush 받아야 COMMIT 반환
- 지연 ↑, 안전 ↑
- `synchronous_commit` 레벨:
  - `off` — local 도 비동기 (위험)
  - `local` — local fsync 만
  - `remote_write` — standby OS 받음
  - `on` (기본) — standby fsync
  - `remote_apply` — standby 가 실제 적용 (read after write 가능)

### 3.3 quorum

```ini
synchronous_standby_names = 'ANY 2 (s1, s2, s3)'    # 임의 2
synchronous_standby_names = 'FIRST 1 (s1, s2)'      # s1 우선
```

---

## 4. Hot Standby — Standby 에서 읽기

```ini
# Standby 의 postgresql.conf
hot_standby = on
hot_standby_feedback = on   # primary 가 vacuum 시 standby 쿼리 보호
```

- Standby 에서 SELECT 가능
- 단점: replication conflict — primary 의 VACUUM 이 standby 의 쿼리 행을 제거 시도 → 쿼리 cancel
  - `hot_standby_feedback = on` 으로 막음 (대신 primary bloat 위험)
  - 또는 `max_standby_streaming_delay = 30s` 로 대기 허용

---

## 5. Logical Replication

### 5.1 구조

```
Publisher (Primary)
  Decode WAL → row 변경 추출 (INSERT/UPDATE/DELETE)
  ↓
Subscriber (다른 PG)
  적용
```

### 5.2 특징

| 특징 | Streaming | Logical |
| --- | --- | --- |
| 단위 | WAL byte | row |
| 버전 차이 | 같아야 | 다를 수 있음 |
| 부분 복제 | ❌ (전체) | ✅ (테이블 / 행 / 컬럼) |
| 쓰기 | Standby read-only | Subscriber 도 쓰기 가능 |
| DDL | 자동 | 수동 |
| Major version 업그레이드 | ❌ | ✅ |

### 5.3 설정

```ini
wal_level = logical
max_replication_slots = 10
max_wal_senders = 10
```

```sql
-- Publisher
CREATE PUBLICATION my_pub FOR TABLE users, orders;
-- 또는 FOR ALL TABLES;

-- Subscriber
CREATE SUBSCRIPTION my_sub
  CONNECTION 'host=publisher dbname=app user=repl password=...'
  PUBLICATION my_pub;
```

### 5.4 행 / 컬럼 필터 (PG 15+)

```sql
CREATE PUBLICATION p FOR TABLE users (id, email) WHERE (active);
```

### 5.5 한계

- PK / Replica Identity 필요 (UPDATE/DELETE 식별 위해)
- 대용량 초기 동기화는 느림
- 충돌 해결 자동 X — 단방향 권장 (양방향은 외부 도구)
- DDL 미동기 — 컬럼 추가 시 양쪽 모두

---

## 6. Failover (장애 절체)

### 6.1 수동 절체

```bash
# 옛 방식 (deprecated): trigger file
# PG 12+:
pg_ctl promote -D /path/to/standby

# 또는 SQL
SELECT pg_promote();
```

→ Standby 가 Primary 로 승격, WAL 받기 종료, 쓰기 가능.

### 6.2 자동 절체 도구

| 도구 | 특징 |
| --- | --- |
| **Patroni** | etcd / consul + 자동 failover. 표준 |
| **repmgr** | EDB 의 전통 도구 |
| **pg_auto_failover** | Citus / Microsoft |
| **Stolon** | Sorintlab |
| **PgPool-II** | + 커넥션 풀 + LB |

Patroni 가 사실상 표준 (Kubernetes 친화).

### 6.3 Split brain
네트워크 분할 시 양쪽이 Primary 라고 생각 → 데이터 분기. 해결: **Quorum 기반 합의** (Patroni + etcd).

---

## 7. Connection Pool 과의 조합

```
Client → HAProxy / PgBouncer → Primary (write)
                              → Standby pool (read)
```

`pg_is_in_recovery()` 로 표적 검사. 또는 `target_session_attrs=read-write` (libpq).

---

## 8. PITR (Point-In-Time Recovery)

WAL Archiving + Base Backup → 특정 시점으로 복구.

### 8.1 설정

```ini
archive_mode = on
archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'
restore_command = 'cp /archive/%f %p'
```

### 8.2 Base Backup

```bash
pg_basebackup -D /backup/base -F tar -z -P
```

### 8.3 복구

```bash
# 새 PGDATA 에 base 풀고
recovery.signal 생성
postgresql.conf 에 추가:
  restore_command = 'cp /archive/%f %p'
  recovery_target_time = '2026-05-13 12:00:00+09'
```

자세히 → [[backup-recovery]]

---

## 9. 모니터링

```sql
-- 복제 지연
SELECT
  application_name,
  pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replay_lag_bytes,
  EXTRACT(EPOCH FROM (now() - reply_time)) AS reply_lag_sec
FROM pg_stat_replication;

-- Slot 의 보존 WAL 크기
SELECT slot_name, active,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained
FROM pg_replication_slots;
```

---

## 10. 함정

### 함정 1 — Replication slot 의 WAL 누적
Standby 가 죽었는데 slot 활성 → Primary 디스크 풀. 알람 + `max_slot_wal_keep_size` (PG 13+).

### 함정 2 — Sync replication 의 단점
하나의 sync standby 가 멈추면 Primary 도 멈춤. ANY N + 여러 standby.

### 함정 3 — Standby 에서 쿼리 cancel
Hot standby 쿼리가 갑자기 죽음. `hot_standby_feedback` 또는 `max_standby_streaming_delay`.

### 함정 4 — Logical replication 의 PK 부재
PK 없으면 UPDATE/DELETE 복제 실패. `REPLICA IDENTITY FULL` (느림) 또는 PK 추가.

### 함정 5 — Major version mismatch
Streaming 은 동일 major 필수. Minor 는 OK (rolling upgrade 가능).

### 함정 6 — DDL 누락
Logical 은 DDL 자동 안 함. Subscriber 에 수동 적용 필수.

### 함정 7 — Initial sync 동안의 무리
`COPY` 단계가 대용량 → 부하. `copy_data = false` 후 수동 동기화.

---

## 11. 학습 자료

- PostgreSQL Documentation Ch. 26 (High Availability)
- **PostgreSQL Replication** — Hans-Jürgen Schönig
- **Patroni Documentation** — patroni.readthedocs.io

---

## 12. 관련

- [[backup-recovery]] — PITR / pg_basebackup
- [[configuration]] — wal_level 등
- [[postgresql]] — PostgreSQL hub
