---
title: "MySQL 설정 — my.cnf / 핵심 변수"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T19:20:00+09:00
tags:
  - database
  - mysql
  - configuration
---

# MySQL 설정 — my.cnf

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 핵심 변수 + SQL_MODE |

**[[mysql|↑ MySQL hub]]**

---

## 1. 설정 파일 위치

```bash
mysql --help | grep -A1 'Default options'
# /etc/my.cnf /etc/mysql/my.cnf ~/.my.cnf /usr/etc/my.cnf
```

### 1.1 우선순위 (아래일수록 우선)

```
/etc/my.cnf
/etc/mysql/my.cnf
$MYSQL_HOME/my.cnf
--defaults-extra-file=...
~/.my.cnf
환경변수 / 커맨드라인
```

### 1.2 SET GLOBAL — 런타임

```sql
SET GLOBAL innodb_buffer_pool_size = 4*1024*1024*1024;   -- 일부만 지원
SET PERSIST innodb_buffer_pool_size = 4*1024*1024*1024;  -- MySQL 8.0+ — 영구화
```

---

## 2. 섹션 구조

```ini
[mysqld]      # 서버
[client]      # CLI
[mysql]       # mysql CLI
[mysqldump]   # dump 도구
```

---

## 3. 핵심 변수 — 메모리

### 3.1 innodb_buffer_pool_size (가장 중요)

```ini
innodb_buffer_pool_size = 8G   # RAM 의 50-70%
```

InnoDB 의 메인 캐시 — 데이터 + 인덱스 + 변경 버퍼. 가장 큰 영향.

### 3.2 innodb_buffer_pool_instances

```ini
innodb_buffer_pool_instances = 8   # buffer pool 1GB 당 1개 권장
```

대형 buffer pool 의 lock contention 분산.

### 3.3 innodb_log_buffer_size

```ini
innodb_log_buffer_size = 64M
```

### 3.4 key_buffer_size (MyISAM 만)
InnoDB 만 쓰면 무관. 작게.

```ini
key_buffer_size = 32M
```

### 3.5 query_cache

MySQL 8.0+ **제거됨**. 5.7 도 비활성 권장 — 경합 ↑.

---

## 4. 핵심 변수 — Connection

```ini
max_connections = 200
max_connect_errors = 1000000
thread_cache_size = 50

wait_timeout = 28800           # 8 hours
interactive_timeout = 28800
```

너무 큰 `max_connections` 는 메모리 / context switch 폭증. **ProxySQL / PgBouncer 류** 사용 권장.

---

## 5. InnoDB 로그 / 동기화

### 5.1 innodb_log_file_size

```ini
innodb_log_file_size = 1G       # 5.7 이전
innodb_redo_log_capacity = 4G   # 8.0.30+
```

크면 crash recovery 시간 ↑, 쓰기 부담 분산 ↓ (좋음). 권장 2-8GB.

### 5.2 innodb_flush_log_at_trx_commit

| 값 | 동작 | Durability | 성능 |
| --- | --- | --- | --- |
| `1` (기본) | 매 커밋 fsync | ★★★ ACID | 표준 |
| `2` | 매 커밋 write (OS), 1초마다 fsync | ★★ (OS crash 시 1초 손실) | 빠름 |
| `0` | 1 초마다 write+fsync | ★ (MySQL crash 시 1초 손실) | 가장 빠름 |

운영 = `1`. 분석 / 임시 = `2` 검토.

### 5.3 sync_binlog

| 값 | 동작 |
| --- | --- |
| `1` (기본) | 매 binlog write fsync — 안전 |
| `0` | OS 에 맡김 — 위험 |
| `N` | N 트랜잭션마다 |

운영 = `1`.

### 5.4 innodb_flush_method

```ini
innodb_flush_method = O_DIRECT     # Linux 권장 (double buffer 회피)
```

---

## 6. SQL 모드

```sql
SHOW VARIABLES LIKE 'sql_mode';
```

MySQL 8.0 기본:
```
ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,
NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
```

| 모드 | 의미 |
| --- | --- |
| `ONLY_FULL_GROUP_BY` | GROUP BY 안 한 컬럼은 SELECT X (SQL 표준) |
| `STRICT_TRANS_TABLES` | 잘못된 값 에러 (조용히 변환 X) |
| `NO_ZERO_DATE` | `'0000-00-00'` 금지 |
| `NO_ZERO_IN_DATE` | `'2026-00-01'` 금지 |
| `ANSI_QUOTES` | `"` 를 식별자로 |
| `PIPES_AS_CONCAT` | `||` 를 concat 으로 |

⚠️ 옛 코드 호환 위해 끄는 경우 있지만 신규는 기본 유지.

---

## 7. 로그

### 7.1 일반 / 오류 로그

```ini
log_error = /var/log/mysql/error.log

# general log — 모든 쿼리. 운영 절대 X (디스크 폭주)
general_log = 0
```

### 7.2 Slow Query Log (필수)

```ini
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1.0
log_queries_not_using_indexes = 1
log_slow_admin_statements = 1
log_slow_replica_statements = 1
```

분석:
```bash
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log
pt-query-digest /var/log/mysql/slow.log   # Percona Toolkit
```

### 7.3 binlog

```ini
log_bin = /var/lib/mysql/mysql-bin
binlog_format = ROW
binlog_expire_logs_seconds = 604800   # 7 일
binlog_row_image = MINIMAL
gtid_mode = ON
enforce_gtid_consistency = ON
```

---

## 8. 인코딩 / 시간대

```ini
character-set-server = utf8mb4
collation-server     = utf8mb4_0900_ai_ci
default-time-zone    = '+00:00'

[client]
default-character-set = utf8mb4
```

---

## 9. 보안

```ini
skip-name-resolve            # DNS lookup 회피 — 빠름
local-infile = 0             # LOAD DATA LOCAL 차단
bind-address = 127.0.0.1     # 외부 차단 (원할 때만)
require_secure_transport = ON  # TLS 강제 (8.0+)
```

### 9.1 비밀번호 정책 (8.0)

```ini
validate_password.policy = MEDIUM
validate_password.length = 12
```

---

## 10. 권장 시작 my.cnf — 16GB / 4 vCPU

```ini
[mysqld]
# 메모리
innodb_buffer_pool_size       = 10G
innodb_buffer_pool_instances  = 8
innodb_log_buffer_size        = 64M
innodb_redo_log_capacity      = 4G

# I/O
innodb_flush_log_at_trx_commit = 1
sync_binlog                    = 1
innodb_flush_method            = O_DIRECT
innodb_io_capacity             = 2000   # SSD
innodb_io_capacity_max         = 4000

# Connection
max_connections   = 200
thread_cache_size = 50

# 로그
slow_query_log    = 1
long_query_time   = 1.0
log_queries_not_using_indexes = 1

# Binlog
log_bin           = mysql-bin
binlog_format     = ROW
binlog_row_image  = MINIMAL
gtid_mode         = ON
enforce_gtid_consistency = ON

# 인코딩
character-set-server = utf8mb4
collation-server     = utf8mb4_0900_ai_ci

# SQL
sql_mode = ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION

# 보안
skip-name-resolve
local-infile = 0
```

---

## 11. 자주 보는 STATUS / VARIABLES

```sql
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Innodb_buffer_pool_pages_%';
SHOW STATUS LIKE 'Slow_queries';
SHOW STATUS LIKE 'Aborted_%';

SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
SHOW VARIABLES LIKE 'max_connections';
```

---

## 12. 함정

### 함정 1 — Buffer pool 너무 작음
기본 128MB. 운영에 부족. RAM 의 50-70%.

### 함정 2 — `sync_binlog = 0`
복제 / 복구 시 데이터 불일치. 운영은 1.

### 함정 3 — `max_connections` 폭증
1000+ 직결 = 메모리 폭발. ProxySQL / 풀.

### 함정 4 — `query_cache` 켬 (8.0 미만)
경합 ↑ → 오히려 느림. 끔.

### 함정 5 — `general_log` 운영 켬
모든 쿼리 디스크 → 폭주.

### 함정 6 — `O_DIRECT` 의 호환성
일부 파일 시스템 (ZFS) 호환 X. `fsync` 사용.

### 함정 7 — 변경 후 재시작 안 함
일부는 SET GLOBAL 동적, 일부는 재시작. `SET PERSIST` (8.0+) 가 깔끔.

---

## 13. 학습 자료

- **MySQL Reference Manual** — Server System Variables
- **High Performance MySQL** (4th) Ch. 5 (Optimizing Server Settings)
- **Percona MySQL Configuration Reference**

---

## 14. 관련

- [[innodb-engine]] — InnoDB 깊이
- [[performance-tuning]] — 더 깊은 튜닝
- [[mysql]] — MySQL hub
