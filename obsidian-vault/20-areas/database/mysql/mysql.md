---
title: "MySQL (Hub)"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T19:10:00+09:00
tags:
  - database
  - rdb
  - mysql
  - hub
---

# MySQL (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | hub + 12 개 세부 노트 |

**[[../database|↑ database hub]]**

---

## 1. 한 줄 정의

**세계에서 가장 많이 쓰이는 오픈소스 RDB**.
1995 MySQL AB → 2008 Sun → 2010 Oracle. InnoDB 가 사실상 표준 스토리지 엔진.
**LAMP 스택** 의 M. 웹 / SaaS 백엔드의 표준.

---

## 2. 역사

| 연도 | 사건 |
| --- | --- |
| 1995 | MySQL 1.0 (Monty Widenius) |
| 2000 | 3.23 — InnoDB 등장 |
| 2008 | Sun 인수 |
| 2010 | Oracle 인수 |
| 2010 | **MariaDB fork** (Monty) |
| 2013 | 5.6 — GTID, online DDL |
| 2015 | **5.7** — JSON, virtual column, 성능 ↑ |
| 2018 | **8.0** — 윈도우 함수, CTE, 새 옵티마이저, role |
| 2024 | 8.4 LTS |

---

## 3. MySQL 의 특징 (요약)

1. **단순성** — 설치 / 운영이 쉬움
2. **거대한 생태계** — 호스팅, 도구, 문서
3. **InnoDB** — MVCC, row lock, FK, crash recovery
4. **다양한 스토리지 엔진** — InnoDB, MyISAM, Memory, Archive
5. **광범위한 복제** — Statement / Row / Mixed, GTID, Group Replication
6. **호환성** — MariaDB, Percona, Aurora 등 fork 풍부
7. **Aurora / PlanetScale** — 클라우드 변형의 인기

---

## 4. 세부 노트 — 이 폴더의 깊이

| 노트 | 영역 |
| --- | --- |
| [[getting-started]] | 설치 / 초기화 / 첫 연결 |
| [[configuration]] | my.cnf / 핵심 변수 |
| [[data-types]] | 숫자 / 문자 / 날짜 / JSON |
| [[sql-syntax]] | DDL / DML / Window / CTE |
| [[innodb-engine]] | InnoDB 구조 / Buffer Pool / Redo Log |
| [[indexes]] | B-Tree / Hash / FULLTEXT / Spatial / Covering |
| [[transactions]] | ACID / Isolation / 락 / Gap Lock |
| [[explain]] | EXPLAIN / EXPLAIN ANALYZE / Optimizer Trace |
| [[replication]] | Statement / Row / GTID / Group Replication |
| [[backup-recovery]] | mysqldump / Xtrabackup / Binlog PITR |
| [[performance-tuning]] | Buffer Pool / Slow Log / 인덱스 |
| [[security]] | root 분실 복구 / GRANT 실수 / mysql_secure_installation / 외부 노출 |
| [[use-cases]] | 언제 MySQL 을 선택해야 하나 |

---

## 5. MySQL 사용 시나리오

| 시나리오 | 적합 |
| --- | --- |
| 웹 / SaaS / CMS | ✅ 표준 |
| WordPress / Magento / Shopify (자체) | ✅ |
| 단순 OLTP | ✅ |
| Read 많은 워크로드 + 복제 | ✅ |
| AWS Aurora 환경 | ✅ (MySQL 호환) |
| 복잡 분석 / Window 많음 | ⚠️ — 8.0 이상이면 OK |
| 지리 / 벡터 / 풍부한 타입 | ⚠️ — PostgreSQL |
| 트랜잭션 DDL 필요 | ❌ — PG (MySQL 은 DDL 자동 commit) |
| 임베디드 | ❌ — SQLite |

---

## 6. 면접 핵심 질문

1. **InnoDB vs MyISAM**.
2. **MVCC** + **Gap Lock** / Next-Key Lock.
3. **B+ Tree 클러스터드 인덱스** — PK 가 데이터.
4. **Isolation Level** — Repeatable Read 의 함정.
5. **GTID 기반 복제** vs 옛 binlog position.
6. **Binlog 포맷** — Statement / Row / Mixed.
7. **Slow Query Log** 활용.
8. **innodb_buffer_pool_size** 권장.
9. **online DDL** 의 한계.
10. **GROUP BY 의 ONLY_FULL_GROUP_BY**.

---

## 7. 학습 자료

- **High Performance MySQL** (4th) — Schwartz / Zaitsev (필독)
- **MySQL Documentation** — dev.mysql.com/doc
- **Percona Blog** — percona.com/blog (필독)
- **PlanetScale Blog**
- **MySQL Internals Manual**

---

## 8. 관련

- [[../postgresql/postgresql]] — PostgreSQL 비교
- [[../sqlite/sqlite]] — SQLite (MySQL 의 가벼운 사촌)
- [[../database-design/database-design]]
- [[../database-tuning/database-tuning]]
- [[../database|↑ database hub]]
