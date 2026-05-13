---
title: "PostgreSQL 설정 — postgresql.conf / pg_hba.conf"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T18:10:00+09:00
tags:
  - database
  - postgresql
  - configuration
---

# PostgreSQL 설정 — postgresql.conf / pg_hba.conf

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 핵심 파라미터 + 인증 설정 |

**[[postgresql|↑ PostgreSQL hub]]**

---

## 1. 설정 파일 위치

```bash
psql -c "SHOW config_file;"
psql -c "SHOW hba_file;"

# 보통
$PGDATA/postgresql.conf
$PGDATA/pg_hba.conf
$PGDATA/pg_ident.conf
```

### 1.1 우선순위

```
postgresql.auto.conf  (ALTER SYSTEM 으로 생성)
> postgresql.conf
> include 파일
> 환경변수
> 기본값
```

---

## 2. postgresql.conf — 핵심 파라미터

### 2.1 연결 / 리소스

```ini
listen_addresses = '*'          # 외부 접속 허용 (보안 주의)
port = 5432
max_connections = 100           # 너무 크게 X — PgBouncer 와 함께
superuser_reserved_connections = 3
```

### 2.2 메모리 (가장 중요)

```ini
shared_buffers = 4GB            # 전체 RAM 의 25% 권장
work_mem = 16MB                 # per operation, per connection
maintenance_work_mem = 1GB      # VACUUM, CREATE INDEX 용
effective_cache_size = 12GB     # OS + Postgres 캐시 추정 — RAM 의 75%
```

### 2.3 WAL / Checkpoint

```ini
wal_level = replica             # minimal / replica / logical
max_wal_size = 4GB              # checkpoint 간 최대
min_wal_size = 1GB
checkpoint_timeout = 15min
checkpoint_completion_target = 0.9   # checkpoint 부담 분산
wal_compression = on
```

### 2.4 통계 / 자동화

```ini
autovacuum = on
autovacuum_naptime = 1min
autovacuum_vacuum_scale_factor = 0.1   # 테이블 10% 변경 시 VACUUM
autovacuum_analyze_scale_factor = 0.05
default_statistics_target = 100         # ANALYZE 표본 크기
```

### 2.5 로그

```ini
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_min_duration_statement = 1000   # 1 초 이상 쿼리만
log_statement = 'ddl'                # none / ddl / mod / all
log_line_prefix = '%t [%p]: db=%d,user=%u,app=%a,client=%h '
log_lock_waits = on
log_temp_files = 0                   # 모든 임시 파일 로그
```

### 2.6 쿼리 제한

```ini
statement_timeout = 30s              # 너무 긴 쿼리 강제 중단
idle_in_transaction_session_timeout = 5min
lock_timeout = 10s
```

### 2.7 동기화

```ini
synchronous_commit = on              # 기본. off 시 데이터 손실 위험 (성능 ↑)
fsync = on                           # 절대 off X (테스트 외)
full_page_writes = on
```

---

## 3. pg_hba.conf — 호스트 기반 인증

### 3.1 구조

```
# TYPE   DATABASE   USER         ADDRESS         METHOD

local    all        all                          peer
host     all        all          127.0.0.1/32    scram-sha-256
host     all        all          ::1/128         scram-sha-256
host     app        app_rw       10.0.0.0/16     scram-sha-256
host     replication  repl_user  10.0.0.0/16     scram-sha-256
host     all        all          0.0.0.0/0       reject
```

### 3.2 TYPE

| Type | 의미 |
| --- | --- |
| `local` | Unix socket |
| `host` | TCP (SSL 무관) |
| `hostssl` | TCP + SSL 만 |
| `hostnossl` | TCP + 비-SSL 만 |

### 3.3 METHOD

| Method | 용도 |
| --- | --- |
| `trust` | ⚠️ 비밀번호 X. 로컬 개발 외 X |
| `peer` | OS 사용자명 = DB 사용자명 (local 만) |
| `md5` | 구식. **사용 X** |
| `scram-sha-256` | ✅ 현대 표준 |
| `cert` | TLS 클라이언트 인증서 |
| `gss` / `sspi` | Kerberos / Windows |
| `reject` | 무조건 거부 |

### 3.4 순서가 중요
위에서 아래로 매칭 — **첫 일치** 가 적용. `reject` 는 명시적 차단을 위에 두기도.

---

## 4. ALTER SYSTEM — 런타임 변경

```sql
-- 설정 변경 (postgresql.auto.conf 에 기록)
ALTER SYSTEM SET work_mem = '32MB';
SELECT pg_reload_conf();    -- reload 가능한 파라미터

-- 일부는 restart 필요 (shared_buffers, max_connections)
SELECT name, setting, context
FROM pg_settings
WHERE name IN ('shared_buffers', 'work_mem', 'max_connections');
-- context: 'postmaster' = restart 필요
```

### 4.1 세션 / 트랜잭션 단위

```sql
SET work_mem = '256MB';            -- 세션 동안
SET LOCAL work_mem = '256MB';      -- 현재 트랜잭션만

-- 함수 단위
CREATE FUNCTION ... SET work_mem = '256MB' AS ...;
```

---

## 5. 권장 시작 설정 — 8GB / 4 vCPU 서버

```ini
shared_buffers = 2GB
work_mem = 16MB
maintenance_work_mem = 512MB
effective_cache_size = 6GB

max_connections = 100
max_wal_size = 4GB
min_wal_size = 1GB
checkpoint_completion_target = 0.9

random_page_cost = 1.1               # SSD 가정 (HDD 면 4.0)
effective_io_concurrency = 200       # SSD
default_statistics_target = 100

log_min_duration_statement = 500
log_lock_waits = on
```

---

## 6. PgTune

[pgtune.leopard.in.ua](https://pgtune.leopard.in.ua) — 자동 추천. 시작점으로 좋음.

```
DB Version: 16
OS: Linux
DB Type: Web Application / OLTP / Mixed / Desktop
Total Memory: 8 GB
CPUs: 4
Connections: 100
Storage: SSD
```

---

## 7. 보안 설정 체크리스트

- [ ] `listen_addresses` 는 필요한 범위만
- [ ] `pg_hba.conf` 에 `trust` 없음
- [ ] `scram-sha-256` 사용 (md5 X)
- [ ] `ssl = on` + 인증서 발급
- [ ] superuser 는 운영 앱에서 사용 X
- [ ] `statement_timeout` / `idle_in_transaction_session_timeout` 설정
- [ ] `log_connections / log_disconnections` 로 감사
- [ ] 정기 `pg_stat_statements` 검토

---

## 8. 함정

### 함정 1 — `max_connections` 너무 큼
1 connection ≈ 10 MB. 1000 connections = 10 GB. **PgBouncer / pgcat** 사용 권장.

### 함정 2 — `work_mem` × N
`work_mem` 은 connection × 동시 operation. 100 connection × 16MB × 5 ops = 8 GB.

### 함정 3 — `shared_buffers` 너무 큼
RAM 의 25% 권장. OS 캐시도 활용해야 함 (double caching).

### 함정 4 — `synchronous_commit = off`
응답속도 ↑ 이지만 crash 시 마지막 ~200 ms 데이터 손실. 의도적 선택만.

### 함정 5 — `fsync = off`
**절대 사용 X**. 단 1 회 crash 로 데이터 파괴.

### 함정 6 — `pg_hba.conf` 변경 후 reload 안 함
`SELECT pg_reload_conf()` 또는 `pg_ctl reload`.

---

## 9. 관련

- [[performance-tuning]] — 더 깊은 튜닝
- [[replication]] — replication 관련 설정
- [[postgresql]] — PostgreSQL hub
