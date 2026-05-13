---
title: "MySQL — 언제 써야 하나"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T20:10:00+09:00
tags:
  - database
  - mysql
  - use-cases
---

# MySQL — 언제 써야 하나

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 적합 / 부적합 + 비교 |

**[[mysql|↑ MySQL hub]]**

---

## 1. 한 줄

> "**단순함 + 광범위한 생태계** 가 필요할 때 MySQL".

PG 보다 단순한 운영. WordPress / SaaS / Aurora 의 표준.

---

## 2. ✅ 적합

### 2.1 단순 OLTP 웹 / API
- 작은 ~ 중간 규모 SaaS
- 사용자 / 콘텐츠 / 거래

### 2.2 LAMP / WordPress / Magento
- 수십 년 검증된 조합

### 2.3 AWS Aurora 환경
- Aurora MySQL — 분산 스토리지, 5x faster (claims)
- RDS MySQL — 매니지드

### 2.4 PlanetScale (서버리스)
- 글로벌 + branching + Vitess 샤딩

### 2.5 거대 운영 노하우 필요
- DBA 인력 / 도구 / 문서 풍부

### 2.6 Read 많은 워크로드
- 단순 복제 + read replica

### 2.7 분석 / Reporting 가벼움
- 8.0+ Window / CTE / Hash Join

---

## 3. ⚠️ 검토 필요

### 3.1 복잡한 SQL / 분석
- PostgreSQL 이 표준 SQL / Window 가 강함
- ClickHouse / BigQuery 가 OLAP

### 3.2 JSON 중심
- MySQL 의 JSON 도 가능, PG JSONB 가 더 강력

### 3.3 거대 쓰기 / 분산
- Vitess / PlanetScale 또는 Cassandra

### 3.4 지리정보
- 5.7+ Spatial OK, PostGIS 가 우위

### 3.5 풀텍스트 검색
- ngram + FULLTEXT 가능, 큰 규모는 Elasticsearch

---

## 4. ❌ 부적합

### 4.1 임베디드 / 모바일
- SQLite

### 4.2 인메모리 캐시
- Redis / Memcached

### 4.3 그래프 (소셜)
- Neo4j

### 4.4 트랜잭션 DDL 강요
- PostgreSQL (MySQL 의 DDL 은 자동 commit)

### 4.5 Document 자유 스키마 (큰 규모)
- MongoDB

---

## 5. MySQL vs PostgreSQL

자세히 → [[../postgresql/use-cases#5-postgresql-vs-mysql]]

| 항목 | MySQL | PostgreSQL |
| --- | --- | --- |
| 단순성 / 학습 | ★★★★★ | ★★★ |
| 표준 SQL | ★★★ | ★★★★★ |
| 트랜잭션 DDL | ❌ | ✅ |
| Window / CTE | ★★★ (8.0+) | ★★★★★ |
| JSON | OK | 강력 (JSONB) |
| 지리 | OK | 강력 (PostGIS) |
| 벡터 / AI | ❌ | ✅ (pgvector) |
| 운영 노하우 | ★★★★★ | ★★★★ |
| 클라우드 매니지드 | Aurora, PlanetScale | RDS, Aurora, Supabase, Neon |
| 단순 OLTP 성능 | 비슷 | 비슷 |

요약: 단순 + Aurora / PlanetScale + 광범위한 생태계 → MySQL. 복잡 도메인 / 분석 / 확장 타입 → PostgreSQL.

---

## 6. MySQL vs SQLite

| 항목 | MySQL | SQLite |
| --- | --- | --- |
| 형태 | 서버 | 임베디드 (파일) |
| 동시 쓰기 | 다수 | 1 (writer 락) |
| 운영 | 필요 | 0 |
| 적합 | 운영 백엔드 | 모바일 / 데스크탑 / 테스트 |

---

## 7. MySQL vs MariaDB

| 항목 | MySQL (Oracle) | MariaDB |
| --- | --- | --- |
| 라이선스 | GPL (커뮤니티) + Enterprise | GPL |
| 8.0 호환 | 표준 | 호환되지만 일부 차이 |
| 스토리지 엔진 | InnoDB | InnoDB + Aria + ColumnStore |
| 클라우드 | RDS, Aurora | RDS, SkySQL |
| 미래 | Oracle 운명 | 커뮤니티 / 비영리 |

신규는 MySQL 8.0 또는 MariaDB 둘 다 OK. 호환성 주의.

---

## 8. MySQL vs Aurora MySQL

| 항목 | MySQL | Aurora MySQL |
| --- | --- | --- |
| 스토리지 | 단일 디스크 | 분산 6 복사본 |
| 쓰기 | 단일 노드 | 단일 writer (또는 multi-master) |
| 읽기 | replica | reader endpoint |
| 복구 | binlog PITR | continuous, 1-35 일 |
| 비용 | 낮음 | 높음 |
| 운영 | 직접 | 매니지드 |

AWS 라면 Aurora 가 표준. 비용 vs 운영 가치 검토.

---

## 9. MySQL vs PlanetScale

| 항목 | MySQL | PlanetScale |
| --- | --- | --- |
| 형태 | 서버 | Vitess 서버리스 |
| 샤딩 | 외부 / 수동 | 자동 (Vitess) |
| Branching | ❌ | ✅ (git-like) |
| FK | ✅ | ❌ (Vitess 제약) |
| Schema migration | 직접 | deploy request |
| 글로벌 | 직접 | 자동 |

신규 / 클라우드 / 수평 확장 가능성 → PlanetScale 검토.

---

## 10. 결정 트리

```
단순 OLTP + 매니지드?
├── AWS  → Aurora MySQL / RDS
├── GCP  → Cloud SQL MySQL
├── 서버리스 → PlanetScale

복잡 도메인 / 분석 / JSON / 지리 / 벡터?
├── PostgreSQL

임베디드?
├── SQLite

검색 / 거대 풀텍스트?
├── Elasticsearch

캐시?
├── Redis

거대 쓰기 분산?
├── Cassandra / Scylla

기본값: MySQL 8.0
```

---

## 11. 회사 사례

| 회사 | MySQL 사용 |
| --- | --- |
| Facebook | 거대 MySQL fleet (MyRocks 자체 포크) |
| GitHub | Vitess / MySQL |
| YouTube | Vitess (MySQL 기반 발명) |
| Uber | MySQL → Schemaless → MySQL |
| Slack | MySQL (Vitess) |
| Booking.com | MySQL |
| Wikipedia | MariaDB |

---

## 12. 관련

- [[../postgresql/use-cases]] — PostgreSQL 비교
- [[../sqlite/use-cases]] — SQLite 비교
- [[mysql]] — MySQL hub
- [[../database]] — database hub
