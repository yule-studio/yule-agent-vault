---
title: "PostgreSQL 백업 / 복구 — pg_dump / pg_basebackup / PITR"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T18:50:00+09:00
tags:
  - database
  - postgresql
  - backup
  - recovery
---

# PostgreSQL 백업 / 복구

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 논리 / 물리 / PITR |

**[[postgresql|↑ PostgreSQL hub]]**

---

## 1. 백업 종류

| 종류 | 도구 | 단위 | 복구 시점 |
| --- | --- | --- | --- |
| **논리** | `pg_dump`, `pg_dumpall` | SQL / 커스텀 | 백업 시점 |
| **물리** | `pg_basebackup` | 파일 시스템 | 백업 시점 |
| **PITR** | base backup + WAL archive | 파일 + WAL | **임의 시점** |
| **스냅샷** | LVM / EBS snapshot | 디스크 | 스냅샷 시점 |

---

## 2. pg_dump — 논리 백업

### 2.1 단일 DB

```bash
# SQL 텍스트
pg_dump -h host -U app -d appdb -f appdb.sql

# 커스텀 포맷 (압축, 병렬 가능)
pg_dump -h host -U app -d appdb -Fc -f appdb.dump

# 디렉터리 (병렬)
pg_dump -h host -U app -d appdb -Fd -j 4 -f appdb-dir
```

### 2.2 옵션

| 옵션 | 의미 |
| --- | --- |
| `-Fc` | 커스텀 포맷 (압축, 선택 복원) |
| `-Fd` | 디렉터리 (병렬) |
| `-Ft` | tar |
| `-Fp` | 일반 SQL (기본) |
| `-j N` | 병렬 worker |
| `-s` | 스키마만 |
| `-a` | 데이터만 |
| `-t tablename` | 특정 테이블만 |
| `-n schemaname` | 특정 스키마만 |
| `--exclude-table` | 제외 |
| `-Z 6` | gzip 압축 레벨 |

### 2.3 전체 클러스터

```bash
pg_dumpall -h host -U postgres -f all.sql
# 사용자 / 권한 / 데이터베이스 모두
# globals 만:
pg_dumpall -g -f globals.sql
```

### 2.4 복원

```bash
# 일반 SQL
psql -d appdb -f appdb.sql

# 커스텀 / 디렉터리
pg_restore -d appdb -j 4 appdb.dump
pg_restore -d appdb -j 4 appdb-dir/

# 일부만
pg_restore -d appdb -t users appdb.dump
pg_restore -l appdb.dump   # 목차 보기
```

### 2.5 장단점

✅ 버전 / 아키텍처 무관, 일부 복원, 작은 DB 에 좋음
❌ 큰 DB 에 느림, 일관성 시점이 dump 시작 시점

---

## 3. pg_basebackup — 물리 백업

```bash
pg_basebackup \
  -h primary -U repl_user \
  -D /backup/base-$(date +%Y%m%d) \
  -Ft -z -P \
  -X stream
```

| 옵션 | 의미 |
| --- | --- |
| `-D` | 출력 디렉터리 |
| `-Ft` | tar 포맷 |
| `-z` | gzip 압축 |
| `-P` | 진행 표시 |
| `-X stream` | WAL 동시 스트림 (별도 connection) |
| `-R` | 복구용 설정 자동 생성 |
| `-c fast` | 즉시 checkpoint |

### 3.1 복원

```bash
# 1. 새 PGDATA 디렉터리에 tar 풀기
mkdir -p /var/lib/postgresql/16/main
tar -xzf /backup/base/base.tar.gz -C /var/lib/postgresql/16/main

# 2. 권한
chown -R postgres:postgres /var/lib/postgresql/16/main
chmod 700 /var/lib/postgresql/16/main

# 3. 시작
systemctl start postgresql
```

---

## 4. PITR (Point-In-Time Recovery)

WAL Archiving + Base Backup → 임의 시점 복구.

### 4.1 설정

```ini
# postgresql.conf
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'
# 또는
archive_command = 'aws s3 cp %p s3://my-bucket/wal/%f'
```

### 4.2 베이스 백업

```bash
pg_basebackup -D /backup/base-20260513 -Ft -z -P -X none
# WAL 은 archive 로 따로 들어옴
```

### 4.3 복구 절차

```bash
# 1. PGDATA 초기화 (또는 새 위치)
rm -rf $PGDATA/*
tar -xzf /backup/base-20260513/base.tar.gz -C $PGDATA
tar -xzf /backup/base-20260513/pg_wal.tar.gz -C $PGDATA/pg_wal

# 2. 복구 설정
cat > $PGDATA/postgresql.auto.conf <<EOF
restore_command = 'cp /archive/%f %p'
recovery_target_time = '2026-05-13 12:34:56+09'
recovery_target_action = 'promote'
EOF

touch $PGDATA/recovery.signal

# 3. 시작
pg_ctl -D $PGDATA start

# 4. PG 가 base + WAL replay → target_time 까지 복구 후 promote
```

### 4.4 recovery target 종류

| 타겟 | 예 |
| --- | --- |
| `recovery_target_time` | `'2026-05-13 12:34:56+09'` |
| `recovery_target_xid` | 트랜잭션 ID |
| `recovery_target_lsn` | WAL 위치 |
| `recovery_target_name` | named restore point |
| `recovery_target = 'immediate'` | 일관 상태 도달 직후 |

```sql
-- named restore point — 백업 전 마킹
SELECT pg_create_restore_point('before_migration');
```

---

## 5. pgBackRest — 운영 권장

```ini
# /etc/pgbackrest.conf
[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2

[main]
pg1-path=/var/lib/postgresql/16/main
```

```bash
pgbackrest --stanza=main backup --type=full
pgbackrest --stanza=main backup --type=incr
pgbackrest --stanza=main info
pgbackrest --stanza=main restore --target-time='...'
```

특징:
- 병렬 백업 / 복원
- 증분 / 차분 / 전체
- 압축 (lz4, zstd)
- S3 / GCS / Azure
- 검증 (`check`)

---

## 6. 기타 도구

| 도구 | 특징 |
| --- | --- |
| **WAL-G** | S3 친화, 증분 |
| **Barman** | 2ndQuadrant, 한 곳에서 여러 PG 관리 |
| **pg_probackup** | Postgres Pro |
| **bman** | Crunchy Data |

---

## 7. 검증 — 백업이 진짜 복원 가능한가

> **테스트 안 한 백업 = 백업 아님**.

### 7.1 정기 복원 훈련

```bash
# Cron: 매주 별도 호스트에 복원 + 쿼리
pg_restore ... && psql -c "SELECT COUNT(*) FROM users"
```

### 7.2 pgBackRest 의 verify

```bash
pgbackrest --stanza=main verify
```

---

## 8. 백업 전략 패턴

### 8.1 작은 DB (< 100GB)

- Daily `pg_dump -Fc` → S3
- 보존 7 일 / 4 주 / 6 개월

### 8.2 중간 DB (100GB ~ 1TB)

- Weekly base + Daily incr (pgBackRest) → S3
- WAL archive 연속

### 8.3 큰 DB (> 1TB)

- pgBackRest 병렬 + zstd
- 또는 EBS snapshot + WAL archive
- RPO / RTO 명시

---

## 9. RPO / RTO

| 지표 | 의미 |
| --- | --- |
| **RPO** (Recovery Point Objective) | 데이터 손실 허용 시간 |
| **RTO** (Recovery Time Objective) | 복구 소요 시간 |

| 방법 | RPO | RTO |
| --- | --- | --- |
| Daily pg_dump | 24 h | 시간 단위 |
| WAL archive + base | 분 단위 | 시간 단위 |
| Streaming replication + failover | 초 단위 | 분 단위 |
| Sync replication | 0 | 분 단위 |

---

## 10. 함정

### 함정 1 — `pg_dump` 일관성
`pg_dump` 는 시작 시 snapshot. 큰 DB 는 dump 중 데이터 변경 — snapshot 은 유지하지만 long-running tx 발생.

### 함정 2 — `archive_command` 실패
실패 시 WAL 누적 → 디스크 풀. 알람 필수.

### 함정 3 — `-X none` 의 위험
`pg_basebackup -X none` 후 WAL archive 가 끊기면 복구 불가. `stream` / `fetch` 권장.

### 함정 4 — 권한 / 소유자
복원된 PGDATA 는 postgres 소유 + 0700.

### 함정 5 — Major version mismatch
물리 백업은 같은 major 만. 다른 버전이면 `pg_dump` 로 마이그레이션.

### 함정 6 — `restore_command` 누락
PITR 시 archive 에 접근 못 하면 실패. 권한 / 경로 확인.

### 함정 7 — 백업 검증 안 함
정기 복원 테스트 없으면 실재 복구 시 실패 발견. 자동화 필수.

---

## 11. 학습 자료

- PostgreSQL Documentation Ch. 25 (Backup and Restore)
- **pgBackRest Documentation** — pgbackrest.org
- **PostgreSQL Backup and Recovery Strategies** — 2ndQuadrant blog

---

## 12. 관련

- [[replication]] — Replication
- [[configuration]] — archive_mode 등
- [[postgresql]] — PostgreSQL hub
