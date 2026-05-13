---
title: "PostgreSQL 시작하기 — 설치 / 초기화 / 첫 연결"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T18:05:00+09:00
tags:
  - database
  - postgresql
  - setup
---

# PostgreSQL 시작하기

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 설치 / 초기화 / 첫 연결 |

**[[postgresql|↑ PostgreSQL hub]]**

---

## 1. 설치

### 1.1 macOS

```bash
brew install postgresql@16
brew services start postgresql@16

# Postgres.app (GUI)
# https://postgresapp.com — 클릭 한 번으로 여러 버전 관리
```

### 1.2 Ubuntu / Debian

```bash
sudo apt update
sudo apt install postgresql-16 postgresql-client-16

sudo systemctl enable --now postgresql
```

### 1.3 Docker

```bash
docker run -d \
  --name pg16 \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=app \
  -p 5432:5432 \
  -v pg16-data:/var/lib/postgresql/data \
  postgres:16
```

### 1.4 관리형 서비스

| 서비스 | 특징 |
| --- | --- |
| AWS RDS Postgres | 표준 RDS |
| AWS Aurora Postgres | 분산 스토리지, 5x 빠름 (claims) |
| GCP Cloud SQL | 표준 매니지드 |
| Supabase | Postgres + Auth + Realtime |
| Neon | 서버리스 + branching |
| Crunchy Bridge | Postgres 전문가 운영 |

---

## 2. 디렉터리 / 클러스터 개념

```
Cluster = 한 PostgreSQL 인스턴스가 관리하는 DB 모음
PGDATA  = 클러스터 데이터 디렉터리

기본:
  /var/lib/postgresql/16/main/    (Linux)
  /opt/homebrew/var/postgresql@16/ (macOS brew)
  /Users/Shared/PostgreSQL/var-16/ (Postgres.app)
```

### 2.1 initdb (수동)

```bash
initdb -D /path/to/data --encoding=UTF8 --locale=en_US.UTF-8

# 이후
pg_ctl -D /path/to/data start
pg_ctl -D /path/to/data stop
```

---

## 3. 첫 접속

### 3.1 psql

```bash
psql -h localhost -U postgres -d postgres
# 비밀번호 입력 (또는 ~/.pgpass)

# 단축
psql postgres   # 로컬 superuser 로
```

### 3.2 ~/.pgpass

```
hostname:port:database:username:password
localhost:5432:*:postgres:secret
```
권한 `chmod 600 ~/.pgpass` 필수.

### 3.3 환경변수

```bash
export PGHOST=localhost
export PGPORT=5432
export PGUSER=app
export PGDATABASE=app
export PGPASSWORD=secret   # 비추 — .pgpass 권장
```

---

## 4. psql 필수 메타 명령

```sql
\l              -- 데이터베이스 목록
\c app          -- DB 전환
\dt             -- 테이블 목록
\d users        -- 테이블 구조
\du             -- 사용자 / role 목록
\dn             -- 스키마 목록
\df             -- 함수 목록
\di             -- 인덱스 목록
\dx             -- 설치된 extension
\timing on      -- 실행 시간 표시
\x on           -- expanded 출력 (세로)
\watch 1        -- 1 초마다 직전 쿼리 반복
\e              -- 외부 에디터로 쿼리 작성
\q              -- 종료
```

---

## 5. 첫 사용자 / DB 만들기

```sql
-- 슈퍼유저로 접속
CREATE USER app WITH PASSWORD 'secret';
CREATE DATABASE app OWNER app;
GRANT ALL PRIVILEGES ON DATABASE app TO app;

-- 또는 CLI
createuser -P app
createdb -O app app
```

### 5.1 권한 분리 (실무 권장)

```sql
-- DDL 용 (마이그레이션)
CREATE USER app_owner WITH PASSWORD '...';
GRANT CONNECT ON DATABASE app TO app_owner;

-- 애플리케이션 (DML 만)
CREATE USER app_rw WITH PASSWORD '...';
GRANT CONNECT ON DATABASE app TO app_rw;
GRANT USAGE ON SCHEMA public TO app_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_rw;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_rw;

-- 읽기 전용 (분석가, BI)
CREATE USER app_ro WITH PASSWORD '...';
GRANT CONNECT ON DATABASE app TO app_ro;
GRANT USAGE ON SCHEMA public TO app_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_ro;
```

---

## 6. 첫 테이블 만들기

```sql
CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    email       TEXT NOT NULL UNIQUE,
    name        TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

INSERT INTO users (email, name)
VALUES ('alice@example.com', 'Alice'),
       ('bob@example.com',   'Bob');

SELECT * FROM users;
```

---

## 7. 연결 문자열

```
postgresql://user:pass@host:port/dbname?sslmode=require

# 예
postgresql://app:secret@localhost:5432/app
postgresql://app:secret@db.example.com/app?sslmode=require&application_name=api
```

| 파라미터 | 의미 |
| --- | --- |
| `sslmode` | disable / prefer / require / verify-ca / verify-full |
| `application_name` | `pg_stat_activity` 에 표시 |
| `connect_timeout` | 초 |
| `options` | `-c statement_timeout=5000` 등 |

---

## 8. 함정

### 함정 1 — postgres 슈퍼유저로 앱 연결
사고의 시작. 항상 권한 분리.

### 함정 2 — `template1` 수정 후 모든 새 DB 에 전파
`template1` 은 새 DB 생성의 템플릿. 잘못 건드리면 재앙.

### 함정 3 — 인코딩 / locale 불일치
`initdb` 시 `--encoding=UTF8 --locale=en_US.UTF-8` 명시. 한글 정렬은 `C.UTF-8` 또는 ICU.

### 함정 4 — `pg_hba.conf` 의 `trust`
`trust` 는 비밀번호 없이 접속. 로컬 개발 외에는 사용 X. 운영은 `scram-sha-256`.

### 함정 5 — 포트 5432 외부 노출
방화벽 / VPC / Security Group 으로 차단. 외부 직접 노출 금지.

---

## 9. 관련

- [[configuration]] — postgresql.conf / pg_hba.conf
- [[sql-syntax]] — SQL 문법
- [[postgresql]] — PostgreSQL hub
