---
title: "SQLite 한계 / 안티패턴"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T23:45:00+09:00
tags:
  - database
  - sqlite
  - limitations
---

# SQLite 한계 / 안티패턴

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 동시성 / 크기 / 기능 한계 |

**[[sqlite|↑ SQLite hub]]**

---

## 1. 한 줄

> "**SQLite 는 서버를 대체하기 위해 만들어진 게 아니다 — `fopen()` 을 대체하기 위해**" — sqlite.org

이 위치 이해가 핵심.

---

## 2. 동시성 — 단일 Writer

```
read N + write 1 (WAL 모드)
read 1 + write 0 (DELETE 모드 — 옛)
```

- 한 시점에 writer 는 **항상 1**.
- 다른 writer 는 `SQLITE_BUSY` → 대기.

### 2.1 영향
- 다중 동시 write 워크로드 X
- 큰 트랜잭션이 길게 lock → 다른 write 대기

### 2.2 대응
- WAL 모드
- `busy_timeout` 5-30s
- application-level write coalescing
- writer 워커 1 + queue
- 큰 트랜잭션 분할

---

## 3. 네트워크 X

```
SQLite 는 로컬 파일 — 네트워크 마운트 (NFS, SMB) 의 락 비신뢰.
```

→ 다중 클라이언트가 같은 DB 에 접속 X (안전).

### 3.1 대안
- 서버 모드 RDB (PG / MySQL)
- **Turso** — libSQL 기반 분산 SQLite
- **rqlite** — Raft 위의 SQLite
- **Litestream** — S3 로 비동기 replication

---

## 4. 권한 / 사용자 X

SQLite 는 권한 시스템 없음. **파일 시스템 권한 = DB 권한**.
다중 사용자 / 역할 분리 → 서버 DB.

---

## 5. 크기 한계 (이론)

| 항목 | 한계 |
| --- | --- |
| DB 파일 크기 | 281 TB (2^48 bytes, default page 4096) |
| 한 행 (BLOB / TEXT) | 1 GB |
| 한 컬럼 수 | 2000 (compile option) |
| INDEX 컬럼 수 | 2000 |
| ATTACH DB 수 | 10 (max 125) |
| 한 SELECT 의 테이블 | 64 |

대부분의 실무엔 충분. 거대 운영 (TB+) 은 검토.

---

## 6. 기능 차이

### 6.1 ALTER TABLE 제한
컬럼 추가 / DROP / RENAME 만. 타입 변경 X → 재생성 패턴.

### 6.2 RIGHT / FULL OUTER JOIN
3.39+ 부터. 이전엔 LEFT 로 회피.

### 6.3 Stored Procedure / 함수
SQL 함수 사용자 정의는 **라이브러리 API** 로 (호스트 언어).
CREATE FUNCTION X.

### 6.4 사용자 / 권한
없음.

### 6.5 외부 키 기본 OFF
`PRAGMA foreign_keys = ON` 매 connection.

### 6.6 Date / Decimal 타입 X
TEXT / INTEGER 로 모방.

### 6.7 LIMIT 의 표준 X
`LIMIT n OFFSET m` 표준이지만 SQL 표준의 `FETCH FIRST` 도 일부 지원.

### 6.8 트랜잭션 격리 변경 X
항상 Serializable. 다른 레벨 X (PG / MySQL 과 다름).

---

## 7. 분산 / 복제 X (기본)

```
다중 머신 / 글로벌 분산 → 별도 솔루션
```

| 솔루션 | 방식 |
| --- | --- |
| **Litestream** | S3 로 WAL streaming → 다른 노드 복원 |
| **LiteFS** | FUSE 기반 분산 |
| **rqlite** | Raft 위의 SQLite |
| **Turso** | libSQL 분산 매니지드 |
| **Cloudflare D1** | SQLite on Workers |

---

## 8. 텍스트 검색 한계

### 8.1 FTS5 의 한국어
기본 `unicode61` tokenizer 는 한국어 형태소 X. 보조 분석 / 외부 처리 필요.

### 8.2 거대 텍스트 검색
ES 가 우위.

---

## 9. 안티패턴

### 9.1 다중 동시 writer 가정
신규 사용자가 가장 흔히 함. 단일 writer 인 점 명심.

### 9.2 매우 긴 트랜잭션
다른 write 차단. 짧게 / batch.

### 9.3 큰 BLOB 저장
백업 부담. 파일 + 경로 권장.

### 9.4 네트워크 마운트
잠금 비신뢰 → 데이터 손상.

### 9.5 다중 어플리케이션 인스턴스가 같은 파일
좋은 마이그레이션 / locking 없으면 위험.

### 9.6 `AUTOINCREMENT` 남용
대부분 `INTEGER PRIMARY KEY` 면 충분.

### 9.7 풍부한 권한 기대
앱 레이어에서 권한 처리.

### 9.8 운영 OLTP 큰 규모
RAM 캐시 + WAL 로 작은~중간 OK. 거대 (수만 TPS) X.

### 9.9 PRAGMA 안 설정
WAL / FK / busy_timeout 없으면 표준 미달.

---

## 10. 어디까지 가능한가 — 실제 운영 한계

### 10.1 작은 / 중간 웹사이트
- 동시 사용자 100-1000+ 가능 (WAL)
- write QPS 100-1000+
- read QPS 수만 (캐시)

### 10.2 모바일 / 데스크탑
- 표준
- 한 앱 = 한 DB

### 10.3 데이터 분석 / 노트북
- 수 GB ~ 수십 GB OK
- 분석은 ClickHouse / DuckDB 와 결합

---

## 11. DuckDB 와 비교

DuckDB = SQLite 의 OLAP 자매 — 컬럼 저장 + 분석 최적화.

| 항목 | SQLite | DuckDB |
| --- | --- | --- |
| 저장 | row | column |
| OLTP / OLAP | OLTP | OLAP |
| 쿼리 | 단순 / point | 거대 분석 |
| 동시 쓰기 | 1 | 1 (같음) |
| 인기 | 표준 | 새 분석 표준 |

분석은 DuckDB, 운영 KV/OLTP 는 SQLite.

---

## 12. 함정

### 12.1 "임베디드 RDB 니까 모든 기능"
서버 RDB 와 격차 명확. 단일 writer / 권한 X.

### 12.2 "MySQL 처럼 동시 처리"
WAL 도 single writer.

### 12.3 "네트워크 마운트 OK"
NFS / Samba 위험.

### 12.4 "FK 자동"
수동 활성.

### 12.5 "DECIMAL"
없음. 정수 (cent).

---

## 13. 학습 자료

- **Appropriate Uses For SQLite** — sqlite.org/whentouse.html
- **When SQLite is Not the Right Choice**
- **Maximum Number Of Things In SQLite** — sqlite.org/limits.html

---

## 14. 관련

- [[transactions-wal]] — 동시성
- [[use-cases]] — 언제 쓰나
- [[sqlite]] — SQLite hub
