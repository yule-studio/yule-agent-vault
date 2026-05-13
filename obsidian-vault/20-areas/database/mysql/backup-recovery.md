---
title: "MySQL 백업 / 복구 — mysqldump / Xtrabackup / Binlog PITR"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:00:00+09:00
tags:
  - database
  - mysql
  - backup
  - recovery
---

# MySQL 백업 / 복구

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | mysqldump / mysqlpump / Xtrabackup / PITR |

**[[mysql|↑ MySQL hub]]**

---

## 1. 백업 종류

| 종류 | 도구 | 단위 | 잠금 |
| --- | --- | --- | --- |
| **논리** | `mysqldump`, `mysqlpump`, `MySQL Shell util.dumpInstance` | SQL | InnoDB = 거의 X (--single-transaction) |
| **물리** | `xtrabackup`, `mysqlbackup` (Enterprise) | 파일 | Hot backup |
| **스냅샷** | LVM / EBS | 디스크 | 빠르지만 일관성 주의 |
| **PITR** | 위 + binlog | base + binlog | 임의 시점 |

---

## 2. mysqldump

### 2.1 기본

```bash
# 단일 DB
mysqldump -u root -p \
  --single-transaction \
  --routines --events --triggers \
  --hex-blob \
  --set-gtid-purged=OFF \
  app > app.sql

# 전체
mysqldump -u root -p \
  --all-databases \
  --single-transaction \
  --routines --events --triggers \
  > all.sql
```

### 2.2 옵션 의미

| 옵션 | 의미 |
| --- | --- |
| `--single-transaction` | InnoDB consistent snapshot (no lock) |
| `--lock-tables` | MyISAM / 혼합 시 |
| `--master-data=2` | binlog position 을 SQL 코멘트로 |
| `--source-data=2` | 8.0+ 표기 |
| `--gtid` | GTID 포함 |
| `--routines` | 프로시저 / 함수 |
| `--events` | 이벤트 스케줄러 |
| `--triggers` | 트리거 |
| `--hex-blob` | 바이너리를 hex 로 |
| `--no-data` | 스키마만 |
| `--no-create-info` | 데이터만 |
| `--tab` | 테이블당 별도 |

### 2.3 단점
- 큰 DB 에 느림 (단일 thread)
- 복원도 느림
- SQL 텍스트 — 효율 낮음

### 2.4 복원

```bash
mysql -u root -p app < app.sql
```

---

## 3. mysqlpump

병렬 dump (MySQL 5.7+). 하지만 8.0 에서는 **MySQL Shell** 이 권장.

```bash
mysqlpump --default-parallelism=4 -u root -p > dump.sql
```

---

## 4. MySQL Shell util — 8.0+ 권장

```js
// mysqlsh
util.dumpInstance('/backup/full', {
  threads: 8,
  compression: 'zstd',
  consistent: true
})

util.dumpSchemas(['app'], '/backup/app', {threads: 8})
util.dumpTables('app', ['users'], '/backup/users')

// 복원
util.loadDump('/backup/full', {threads: 8, loadUsers: true})
```

특징:
- 병렬
- 압축 (zstd, gzip)
- S3 / OCI 호환 스토리지
- 일관성 보장

---

## 5. Percona Xtrabackup (xtrabackup)

물리 백업 (innodb 데이터 파일 복사) — **hot backup** (서비스 중단 X).

### 5.1 설치 / 백업

```bash
# Percona repo 설치
apt install percona-xtrabackup-80

xtrabackup --backup --target-dir=/backup/$(date +%Y%m%d) \
  --user=root --password=... --parallel=8

# prepare (필수)
xtrabackup --prepare --target-dir=/backup/20260513

# 복원
systemctl stop mysql
rm -rf /var/lib/mysql/*
xtrabackup --copy-back --target-dir=/backup/20260513
chown -R mysql:mysql /var/lib/mysql
systemctl start mysql
```

### 5.2 증분 백업

```bash
# 전체
xtrabackup --backup --target-dir=/backup/full

# 증분 (전체 대비)
xtrabackup --backup --target-dir=/backup/incr1 \
  --incremental-basedir=/backup/full

# prepare 시 합치기
xtrabackup --prepare --apply-log-only --target-dir=/backup/full
xtrabackup --prepare --apply-log-only \
  --target-dir=/backup/full \
  --incremental-dir=/backup/incr1
xtrabackup --prepare --target-dir=/backup/full
```

### 5.3 스트리밍 / 압축 / 암호화

```bash
xtrabackup --backup --stream=xbstream --compress | \
  ssh remote "cat > /backup/backup.xbstream"
```

### 5.4 장점
- Hot, 큰 DB 도 OK
- 증분
- TB+ 규모 표준

---

## 6. PITR — Point-In-Time Recovery

base backup + binlog replay → 임의 시점.

### 6.1 절차

```bash
# 1. base 복원 (mysqldump 또는 xtrabackup)
mysql < base.sql
# 또는
xtrabackup --copy-back --target-dir=/backup/base

# base 시점의 binlog position 확인
# (--source-data=2 으로 dump 한 경우 코멘트에 있음)
# 또는 xtrabackup_info 파일

# 2. binlog 적용
mysqlbinlog \
  --start-position=NNN \
  --stop-datetime='2026-05-13 12:34:56' \
  mysql-bin.000123 mysql-bin.000124 \
  | mysql -u root -p
```

### 6.2 GTID 로 stop 지정

```bash
mysqlbinlog \
  --include-gtids='uuid:1-1000' \
  --exclude-gtids='uuid:500-510' \
  ...
```

특정 잘못된 트랜잭션만 제외하고 복구 — 운영 사고 흔한 시나리오.

---

## 7. binlog 백업

binlog 자체도 백업 — PITR 의 핵심.

```bash
# mysqlbinlog 원격 streaming
mysqlbinlog \
  --read-from-remote-server \
  --raw \
  --host=primary \
  --user=repl --password=... \
  --to-last-log \
  --stop-never \
  --result-file=/backup/binlog/ \
  mysql-bin.000001 &
```

→ 1초 단위로 백업 디렉터리에 binlog 적재.

---

## 8. 클라우드 매니지드

### 8.1 RDS / Aurora
- 자동 daily snapshot + 5분 단위 transaction log
- PITR → 콘솔에서 클릭
- Retention 1-35 일

### 8.2 PlanetScale
- branching 으로 효과적 백업
- Restore 강력

→ 매니지드 사용 시 백업 도구 직접 운영 불필요. 단, 정기 export (S3) 권장.

---

## 9. 검증

> **테스트 안 한 백업 = 백업 아님**.

### 9.1 자동 복원 테스트

```bash
# Cron — 매주 다른 호스트에 복원
xtrabackup --copy-back --target-dir=/backup/latest --datadir=/tmp/test
mysqld --datadir=/tmp/test &
mysql -e "SELECT COUNT(*) FROM app.users;"
```

### 9.2 mysqlcheck

```bash
mysqlcheck --check --all-databases --user=root -p
```

---

## 10. 보존 / 정책

| 패턴 | 보존 |
| --- | --- |
| Daily | 7 일 |
| Weekly | 4 주 |
| Monthly | 12 개월 |
| Yearly | 7+ 년 (규제) |

binlog: 7-14 일이 일반적 (PITR 윈도우).

---

## 11. 백업 전략 예

### 11.1 작은 DB (< 100GB)
- Nightly `MySQL Shell util.dumpInstance` → S3
- binlog 14 일 보존
- RPO 5분 (binlog 스트리밍), RTO 1-2 시간

### 11.2 중간 DB (100GB ~ 1TB)
- Weekly Xtrabackup full + Daily incremental → S3
- binlog 7 일
- RPO 1분, RTO 시간 단위

### 11.3 큰 DB (> 1TB)
- Xtrabackup 증분 + zstd
- binlog 실시간 stream
- RPO 초 단위, RTO 분 단위 (replica 자동 promote)

---

## 12. 함정

### 함정 1 — `mysqldump` 의 lock
`--single-transaction` 없으면 테이블 lock. 큰 DB 에 치명적.

### 함정 2 — `--single-transaction` + DDL
백업 중 DDL → 일관성 깨짐. 백업 윈도우 동안 DDL 금지.

### 함정 3 — Xtrabackup `prepare` 누락
prepare 안 하면 crash recovery 미완 → 시작 안 됨.

### 함정 4 — binlog 만으로 PITR 불가
반드시 base + binlog. base 없으면 무용.

### 함정 5 — `--gtid` / `--source-data` 미사용
복제 / PITR 기준점 없음.

### 함정 6 — 같은 디스크에 백업
디스크 장애 = 백업도 사라짐. **별도 스토리지** 필수.

### 함정 7 — 복원 미테스트
정기 훈련 안 하면 실전 실패 발견.

### 함정 8 — 패스워드 / 권한 누락
백업에 user 정의 / 권한 미포함 — `--all-databases` 또는 `mysql` system DB 포함.

---

## 13. 학습 자료

- **MySQL Reference Manual** Ch. 7 (Backup and Recovery)
- **Percona Xtrabackup Documentation**
- **MySQL Shell Utilities** — dev.mysql.com/doc/mysql-shell

---

## 14. 관련

- [[replication]] — binlog / GTID
- [[configuration]] — binlog 설정
- [[mysql]] — MySQL hub
