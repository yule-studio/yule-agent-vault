---
title: "SQLite — 언제 써야 하나"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T23:50:00+09:00
tags:
  - database
  - sqlite
  - use-cases
---

# SQLite — 언제 써야 하나

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 적합 / 부적합 + 비교 |

**[[sqlite|↑ SQLite hub]]**

---

## 1. 한 줄

> "**임베디드 / 로컬 / 단일 파일** 이면 SQLite".

`fopen()` 의 대체. 서버 DB 의 대안이 아니라 **파일 형식의 대안**.

---

## 2. ✅ 적합

### 2.1 모바일 (iOS / Android)
- 모든 모바일 앱의 표준 로컬 DB
- Core Data, Room 모두 내부적으로 SQLite

### 2.2 데스크탑 앱
- 브라우저 (Chrome / Firefox / Safari 모두)
- IDE / 에디터
- 메신저 / 메일

### 2.3 IoT / 임베디드
- 작은 메모리에서 동작
- 한 디바이스 한 DB

### 2.4 테스트 / 개발
- in-memory 또는 임시 파일
- 빠른 setup / teardown

### 2.5 데이터 분석 / 노트북
- Jupyter / Pandas 와 함께
- CSV 보다 강력

### 2.6 로컬 캐시
- 앱의 로컬 캐시 / offline mode

### 2.7 작은 ~ 중간 웹사이트 (WAL)
- 단일 writer 한계 안에서
- 운영 사례 늘어남 (이메일 / 블로그 등)

### 2.8 Edge / 매니지드 (Turso, D1, Litestream)
- 새 트렌드 — 글로벌 분산 SQLite

### 2.9 설정 / 사용자 데이터
- 단일 사용자의 설정 / preference

### 2.10 파일 포맷
- .sqlite 파일을 데이터 교환 포맷으로
- Application 의 document format

---

## 3. ⚠️ 검토

### 3.1 작은 멀티 유저 웹사이트
- WAL + 단일 writer 라면 트래픽 작을 때 가능
- 안전하게 = PG / MySQL

### 3.2 분석 (column scan)
- DuckDB 가 우위

### 3.3 풀텍스트 (한국어, 큰)
- FTS5 + ngram OK 작은 / 단순
- 큰 / 한국어 형태소 → ES

---

## 4. ❌ 부적합

### 4.1 다중 동시 writer
- 단일 writer 한계
- PG / MySQL / 분산

### 4.2 네트워크 / 다중 머신
- 분산 X (외부 솔루션 없이는)

### 4.3 권한 / 다중 사용자
- DB 권한 X
- 서버 DB

### 4.4 거대 운영 (TB+ + 거대 TPS)
- 한계 도달

---

## 5. SQLite vs PostgreSQL

| 항목 | SQLite | PostgreSQL |
| --- | --- | --- |
| 형태 | 임베디드 (파일) | 서버 |
| 운영 | 0 | 필요 |
| 동시 쓰기 | 1 | 다수 |
| 권한 | X | 강 |
| 네트워크 | X | ✅ |
| 풍부한 기능 | 작음 | 매우 풍부 |
| 학습 곡선 | 매우 낮음 | 중 |
| 가격 | 무료 + 운영 0 | 무료 + 운영 |

---

## 6. SQLite vs MySQL / MariaDB

비슷한 비교. SQLite = local, MySQL = server.

---

## 7. SQLite vs DuckDB

| 항목 | SQLite | DuckDB |
| --- | --- | --- |
| 저장 | row | column |
| 워크로드 | OLTP / KV | OLAP / 분석 |
| 쿼리 | point / 작은 range | 거대 scan / agg |
| 형태 | 임베디드 | 임베디드 |
| 동시 쓰기 | 1 | 1 |
| 인기 | 표준 | 새 분석 표준 |

분석 → DuckDB. OLTP / KV → SQLite. 둘 다 같은 앱에서 쓰기도.

---

## 8. SQLite vs RocksDB / LevelDB

| 항목 | SQLite | RocksDB |
| --- | --- | --- |
| SQL | ✅ | ❌ (KV) |
| 형태 | 단일 파일 | LSM 디렉터리 |
| 동시 쓰기 | 1 | 다수 |
| 거대 쓰기 | 한계 | 매우 강 |
| 사용 | 일반 | 인프라 / DB 엔진의 백엔드 |

RocksDB 는 다른 DB (Cassandra, CockroachDB 등) 의 백엔드. 일반 앱은 SQLite.

---

## 9. SQLite vs Turso / libSQL

Turso = libSQL (SQLite fork) + 매니지드 분산 / Edge.

| 항목 | SQLite | Turso |
| --- | --- | --- |
| 형태 | 로컬 | 분산 / Edge |
| 호환 | 표준 | 호환 + 새 기능 |
| 운영 | 0 | 매니지드 |
| 글로벌 | X | ✅ |

작은 Edge 앱 / 글로벌 분산이 필요하면 Turso 검토.

---

## 10. 결정 트리

```
임베디드 / 로컬?
└── SQLite

작은 웹사이트 + WAL + 단일 writer OK?
└── SQLite (with Litestream 백업)

작은 ~ 중간 운영 DB?
└── PostgreSQL / MySQL

분석 / OLAP?
└── DuckDB / ClickHouse

거대 운영?
└── PG / MySQL + Cassandra / Aurora ...
```

---

## 11. 회사 사례

| 회사 | SQLite 사용 |
| --- | --- |
| Apple | Mail, iMessage, Core Data 내부 |
| Google | Android, Chrome |
| Mozilla | Firefox |
| Adobe | Photoshop 일부 |
| Skype | DB |
| Dropbox | 클라이언트 |
| WhatsApp | DB |
| Bloomberg | 시계열 일부 |
| Tesla | 차량 일부 |
| 항공 | 항공기 (DRH 자랑) |

→ 가장 많이 배포된 DB 라는 말은 과장이 아님.

---

## 12. Litestream / LiteFS / Turso — 새 운영 패턴

SQLite 가 운영 DB 로 부활하는 흐름:

```
Litestream: WAL → S3 streaming (replica / DR)
LiteFS:     FUSE 기반 분산 (Fly.io)
Turso:      libSQL 매니지드 / Edge
D1:         Cloudflare 의 매니지드 SQLite
```

→ 작은 운영 (이메일 / 블로그 / 도구) 에 SQLite + S3 backup 으로 충분한 경우 증가.

---

## 13. 관련

- [[../postgresql/use-cases]] — PG 비교
- [[../mysql/use-cases]] — MySQL 비교
- [[sqlite]] — SQLite hub
- [[../database]] — database hub
