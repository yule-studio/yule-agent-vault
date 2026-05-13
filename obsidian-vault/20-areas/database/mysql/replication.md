---
title: "MySQL 복제 — GTID / Row / Group Replication"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T19:55:00+09:00
tags:
  - database
  - mysql
  - replication
  - ha
---

# MySQL 복제

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Binlog / GTID / Group Replication |

**[[mysql|↑ MySQL hub]]**

---

## 1. 복제 종류

| 종류 | 단위 | 용도 |
| --- | --- | --- |
| **Async Replication** (기본) | binlog event | Read scale / DR |
| **Semi-Sync** | + 1 replica ack | 안전 ↑ |
| **Group Replication** | Paxos 합의 | InnoDB Cluster |
| **MySQL NDB** | 클러스터 엔진 | 특수 — 거의 안 씀 |

---

## 2. Binlog — Replication 의 토대

```ini
log_bin = mysql-bin
binlog_format = ROW            # STATEMENT / ROW / MIXED
binlog_row_image = MINIMAL     # FULL / MINIMAL / NOBLOB
sync_binlog = 1
binlog_expire_logs_seconds = 604800   # 7 일
```

### 2.1 binlog_format

| 포맷 | 의미 | 권장 |
| --- | --- | --- |
| STATEMENT | SQL 그대로 | ⚠️ 비결정적 함수 위험 (`NOW()`, `RAND()`) |
| **ROW** | 행 변경 사실 | ✅ 표준 |
| MIXED | 자동 선택 | OK |

### 2.2 binlog 도구

```bash
mysqlbinlog --start-datetime='2026-05-13 00:00:00' mysql-bin.000123
mysqlbinlog --base64-output=DECODE-ROWS -v mysql-bin.000123   # ROW 디코드
```

---

## 3. Async Replication 구성

### 3.1 Primary 설정

```ini
# my.cnf
server_id = 1
log_bin = mysql-bin
binlog_format = ROW
gtid_mode = ON
enforce_gtid_consistency = ON
```

```sql
CREATE USER 'repl'@'%' IDENTIFIED BY '...';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
```

### 3.2 Replica 설정

```ini
server_id = 2
relay_log = mysql-relay
gtid_mode = ON
enforce_gtid_consistency = ON
read_only = ON
super_read_only = ON
```

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='primary-host',
  SOURCE_USER='repl',
  SOURCE_PASSWORD='...',
  SOURCE_AUTO_POSITION=1;     -- GTID 기반

START REPLICA;
SHOW REPLICA STATUS\G
```

⚠️ MySQL 8.0+ 부터 `MASTER` → `SOURCE`, `SLAVE` → `REPLICA` 로 변경.

### 3.3 초기 데이터 복사

```bash
# 옵션 A — mysqldump
mysqldump --source-data=2 --gtid \
  --single-transaction --triggers --routines --events \
  --all-databases > full.sql

# 옵션 B — Xtrabackup (큰 DB 권장)
xtrabackup --backup --target-dir=/backup
xtrabackup --prepare --target-dir=/backup
# 복사 후 시작 + CHANGE REPLICATION SOURCE
```

---

## 4. GTID (Global Transaction ID)

```
GTID = source_uuid:transaction_id
       3e11fa47-71ca-11e1-9e33-c80aa9429562:1-100
```

### 4.1 장점
- 위치 (binlog filename + offset) 추적 불필요
- 자동 failover / 재구성 쉬움
- 중복 / 누락 방지

### 4.2 설정

```ini
gtid_mode = ON
enforce_gtid_consistency = ON
log_replica_updates = ON       # cascading 복제용
```

### 4.3 옛 binlog position vs GTID
새로 시작하면 **무조건 GTID**. 옛 환경만 binlog file/position.

---

## 5. Semi-Synchronous Replication

```sql
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
INSTALL PLUGIN rpl_semi_sync_slave  SONAME 'semisync_slave.so';

SET GLOBAL rpl_semi_sync_master_enabled = 1;
SET GLOBAL rpl_semi_sync_slave_enabled  = 1;
SET GLOBAL rpl_semi_sync_master_timeout = 1000;   -- ms
```

→ Primary 가 binlog 보내고 **최소 1 replica ack** 까지 기다림.
타임아웃 시 async fallback. 비동기보다 안전, 동기보다 빠름.

---

## 6. Replica 모니터링

```sql
SHOW REPLICA STATUS\G
```

핵심:

| 필드 | 의미 |
| --- | --- |
| `Replica_IO_Running` | binlog 수신 | YES |
| `Replica_SQL_Running` | 적용 | YES |
| `Seconds_Behind_Source` | 적용 지연 (초) |
| `Retrieved_Gtid_Set` | 받은 GTID |
| `Executed_Gtid_Set` | 적용된 GTID |
| `Last_Error` | 마지막 에러 |

---

## 7. Replica 에러 처리

```sql
-- 잘못된 이벤트 건너뛰기 (위험)
STOP REPLICA;
SET GLOBAL sql_replica_skip_counter = 1;
START REPLICA;

-- GTID 기반: empty transaction 으로 마킹
STOP REPLICA;
SET GTID_NEXT='source_uuid:N';
BEGIN; COMMIT;
SET GTID_NEXT='AUTOMATIC';
START REPLICA;
```

→ 자주 발생하면 binlog_format / 스키마 차이 / 충돌 점검.

---

## 8. Multi-Source Replication (8.0)

여러 source 의 binlog 를 하나의 replica 에 통합.

```sql
CHANGE REPLICATION SOURCE TO ... FOR CHANNEL 'src1';
CHANGE REPLICATION SOURCE TO ... FOR CHANNEL 'src2';
START REPLICA FOR CHANNEL 'src1';
```

ETL / 통합 reporting 등.

---

## 9. Parallel Replication

```ini
replica_parallel_workers = 8
replica_parallel_type = LOGICAL_CLOCK   # 8.0 기본
binlog_transaction_dependency_tracking = WRITESET  # 더 적극적
```

→ 단일 thread 보다 빠른 적용. 8.0 부터 표준.

---

## 10. Group Replication (InnoDB Cluster)

Paxos 기반 합의 → **multi-primary / single-primary** 가능.

### 10.1 단일-primary (권장)

```ini
plugin_load_add = 'group_replication.so'
group_replication_group_name = 'aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa'
group_replication_local_address = 'host1:33061'
group_replication_group_seeds = 'host1:33061,host2:33061,host3:33061'
group_replication_bootstrap_group = OFF    # 첫 노드만 ON
group_replication_single_primary_mode = ON
```

### 10.2 MySQL Shell / InnoDB Cluster

```js
// mysqlsh
dba.createCluster('myCluster');
cluster.addInstance('user@host2:3306');
cluster.addInstance('user@host3:3306');
cluster.status();
```

### 10.3 MySQL Router
앞단의 라우터 — RW / RO 분리, failover 자동.

### 10.4 한계
- 노드 수 9 까지
- 트랜잭션 / 행 크기 제한
- 멀티-primary 는 충돌 위험 (단일-primary 권장)

---

## 11. Failover 도구

| 도구 | 특징 |
| --- | --- |
| **MySQL Router + Group Replication** | 공식 |
| **ProxySQL** | 강력한 라우팅 / 풀 |
| **Orchestrator** | GitHub 의 토폴로지 / 자동 promote |
| **MHA (옛)** | Master High Availability — 거의 안 씀 |

---

## 12. Aurora MySQL / PlanetScale

### 12.1 Aurora
- 분산 스토리지 (6 복사본)
- "5x faster" claim
- writer 1 + readers N
- 백업 / failover 매니지드

### 12.2 PlanetScale
- Vitess 기반 (YouTube 의 샤딩)
- 서버리스, branching
- **FK 미지원** (Vitess 제약)

→ 클라우드 전제이면 강력한 옵션.

---

## 13. CDC — binlog → 이벤트 스트림

| 도구 | 흐름 |
| --- | --- |
| **Debezium** | binlog → Kafka |
| **Maxwell** | binlog → Kafka / Kinesis |
| **Flink CDC** | binlog → Flink |
| **AWS DMS** | binlog → 다른 DB |

→ Event Sourcing / DW 적재 / 마이크로서비스.

---

## 14. 함정

### 함정 1 — `binlog_format = STATEMENT`
`UPDATE ... WHERE NOW() < ...` 같은 비결정적 → 복제 불일치. **ROW** 사용.

### 함정 2 — Replica 에서 직접 쓰기
`read_only = OFF` 면 split-brain. `super_read_only = ON`.

### 함정 3 — `Seconds_Behind_Source` 의 함정
큰 트랜잭션 적용 중에는 비정확. `Executed_Gtid_Set` 비교가 더 정확.

### 함정 4 — GTID 의 빈 트랜잭션 누적
잘못된 SKIP 으로 빈 GTID 누적 → 정리 어려움.

### 함정 5 — 옛 MASTER/SLAVE 명령
8.0 부터 deprecated. SOURCE/REPLICA 사용.

### 함정 6 — `sync_binlog = 0` + crash
binlog 손실 → replica 불일치. 운영은 1.

### 함정 7 — replica 의 PK 부재
ROW 포맷에서 PK 없으면 매번 풀스캔. 모든 테이블에 PK.

### 함정 8 — DDL 동기
DDL 은 binlog 로 그대로 → replica 도 lock. online DDL / pt-osc 권장.

---

## 15. 학습 자료

- **MySQL Reference Manual** Ch. 19 (Replication)
- **High Performance MySQL** Ch. 10 (Replication)
- **Orchestrator Documentation**
- **Debezium Documentation**

---

## 16. 관련

- [[backup-recovery]] — binlog PITR
- [[configuration]] — binlog / GTID
- [[mysql]] — MySQL hub
