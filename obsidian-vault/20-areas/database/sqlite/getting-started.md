---
title: "SQLite 시작하기 — 설치 / CLI / 첫 명령"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T23:20:00+09:00
tags:
  - database
  - sqlite
  - setup
---

# SQLite 시작하기

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 설치 / CLI |

**[[sqlite|↑ SQLite hub]]**

---

## 1. 설치

### 1.1 macOS / Linux
대부분 기본 포함:

```bash
sqlite3 --version
# 없으면
brew install sqlite       # macOS
apt install sqlite3        # Ubuntu
```

### 1.2 Windows
[sqlite.org/download](https://sqlite.org/download.html) — `sqlite-tools-win-*.zip`.

### 1.3 라이브러리 사용 (대부분의 경우)
프로그래밍 언어가 SQLite 를 라이브러리로 포함:

| 언어 | 라이브러리 |
| --- | --- |
| Python | 표준 `sqlite3` |
| Node | `better-sqlite3` (동기, 추천) / `sqlite3` (비동기) |
| Go | `mattn/go-sqlite3` / `modernc.org/sqlite` (cgo X) |
| Rust | `rusqlite` |
| Java | `xerial/sqlite-jdbc` |
| Swift / Kotlin | iOS / Android 기본 |
| Ruby | `sqlite3` gem |

---

## 2. 첫 명령 — CLI

```bash
# 새 파일 (없으면 생성)
sqlite3 app.db

# 또는 메모리만
sqlite3 :memory:
```

```sql
sqlite> .help
sqlite> .tables                 -- 테이블 목록
sqlite> .schema users           -- DDL 보기
sqlite> .databases              -- 연결된 DB
sqlite> .mode column            -- 출력 포맷
sqlite> .headers on
sqlite> .timer on
sqlite> .changes on             -- 변경 행 수 표시
sqlite> .quit

-- SQL
CREATE TABLE users (
  id INTEGER PRIMARY KEY,
  email TEXT NOT NULL UNIQUE,
  name TEXT,
  created_at TEXT DEFAULT (datetime('now'))
);

INSERT INTO users (email, name) VALUES ('a@x.com','Alice');
SELECT * FROM users;
```

---

## 3. 자주 쓰는 CLI 명령

```
.tables             -- 테이블 목록
.schema [name]      -- DDL
.indexes [table]    -- 인덱스
.dump               -- SQL 텍스트로 export
.read file.sql      -- SQL 파일 실행
.import data.csv tname   -- CSV 가져오기
.output file.csv    -- 출력 파일로
.mode csv
SELECT ...;
.output stdout

.shell ls           -- 셸 명령 실행
.system pwd
.cd /tmp
.databases
.fullschema         -- 전체 스키마
.eqp on             -- 매 쿼리 EXPLAIN
.expert             -- 인덱스 추천 (3.32+)
.recover            -- 손상 파일 복구
.backup file.db     -- 백업
.restore file.db    -- 복원
```

---

## 4. 파일 / 메모리

```bash
sqlite3 mydb.db                                   # 파일
sqlite3 :memory:                                  # 메모리 (휘발)
sqlite3 'file:mydb.db?mode=ro'                    # URI — 읽기 전용
sqlite3 'file:/tmp/x.db?cache=shared'             # 공유 캐시
```

---

## 5. 권장 PRAGMA (시작 시)

```sql
PRAGMA journal_mode = WAL;              -- 동시 read + write
PRAGMA synchronous = NORMAL;            -- 균형
PRAGMA temp_store = MEMORY;
PRAGMA mmap_size = 30000000000;         -- 30GB (큰 DB)
PRAGMA cache_size = -64000;             -- 64MB (음수 = KB)
PRAGMA foreign_keys = ON;               -- FK 활성 (기본 OFF!)
PRAGMA busy_timeout = 5000;             -- 5초 대기
```

자세히 → [[configuration]]

---

## 6. 첫 테이블 + 데이터

```sql
CREATE TABLE users (
  id          INTEGER PRIMARY KEY,    -- AUTOINCREMENT 보통 X
  email       TEXT NOT NULL UNIQUE,
  name        TEXT,
  age         INTEGER CHECK (age >= 0),
  created_at  TEXT NOT NULL DEFAULT (datetime('now', 'subsec'))
) STRICT;                                -- STRICT 테이블 (3.37+)

CREATE INDEX idx_users_created ON users(created_at);

INSERT INTO users (email, name, age) VALUES
  ('a@x.com', 'Alice', 30),
  ('b@x.com', 'Bob',   25);

SELECT * FROM users ORDER BY created_at DESC;
```

---

## 7. INTEGER PRIMARY KEY

```sql
id INTEGER PRIMARY KEY            -- rowid 의 별칭, 자동 증가
id INTEGER PRIMARY KEY AUTOINCREMENT   -- + 한 번 쓴 ID 재사용 X (느림)
```

대부분 `INTEGER PRIMARY KEY` 만으로 충분.

---

## 8. 데이터 import / export

### 8.1 CSV → SQLite

```bash
sqlite3 mydb.db <<EOF
.mode csv
.import users.csv users
EOF
```

### 8.2 SQLite → CSV

```bash
sqlite3 mydb.db <<EOF
.mode csv
.headers on
.output users.csv
SELECT * FROM users;
EOF
```

### 8.3 .dump

```bash
sqlite3 mydb.db .dump > backup.sql
sqlite3 newdb.db < backup.sql
```

---

## 9. Python 빠른 예

```python
import sqlite3

conn = sqlite3.connect("app.db")
conn.execute("PRAGMA journal_mode=WAL")
conn.execute("PRAGMA foreign_keys=ON")
conn.row_factory = sqlite3.Row

cur = conn.execute("SELECT * FROM users WHERE age > ?", (18,))
for row in cur:
    print(row["name"], row["age"])

with conn:                                # 자동 commit / rollback
    conn.execute("INSERT INTO users (email,name) VALUES (?,?)",
                 ("c@x.com", "Carol"))
```

---

## 10. 백업

```bash
# 라이브 백업 (다른 연결 차단 X)
sqlite3 app.db ".backup '/backup/app-$(date +%F).db'"

# 또는 cp (read-only 일 때만 안전)
cp app.db backup.db
```

---

## 11. GUI

| 도구 | 특징 |
| --- | --- |
| **DB Browser for SQLite** | 무료, 표준 |
| **DBeaver** | 다중 DB 지원 |
| **TablePlus** | 매끄러움 |
| **DataGrip** | JetBrains |

---

## 12. 함정

### 12.1 FK 기본 OFF
**`PRAGMA foreign_keys = ON`** 매 연결마다 (라이브러리 자동인 경우도).

### 12.2 journal_mode 기본 DELETE
WAL 가 보통 더 좋음. 시작 시 설정.

### 12.3 `AUTOINCREMENT` 사용
대부분 불필요 + 약간 느림. `INTEGER PRIMARY KEY` 만으로 충분.

### 12.4 동시 writer 다수
SQLite 는 single writer. busy timeout / WAL / 재시도.

### 12.5 파일 잠금 — NFS / 클라우드 파일시스템
잠금 비정상. 로컬 파일 시스템.

### 12.6 큰 BLOB
파일을 DB 안에 — 가능하지만 큰 DB 는 백업 / 복제 부담.

---

## 13. 관련

- [[configuration]] — PRAGMA
- [[sql-syntax]]
- [[transactions-wal]]
- [[sqlite]] — SQLite hub
