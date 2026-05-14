---
title: "GCP Spanner — Globally Distributed RDB"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:45:00+09:00
tags:
  - gcp
  - database
  - spanner
---

# GCP Spanner

**[[database|↑ Database]]** · **[[../cloud-gcp|↑↑ GCP]]**

---

## 1. 한 줄

Google 의 **글로벌 분산 RDB** — 강한 일관성 + 99.999% SLA + 무한 수평 확장 + SQL.
2017 GA. AdWords / Spotify / Niantic / 거대 SaaS 의 backbone.

---

## 2. 왜

- 일반 RDB = 단일 노드 (Aurora 도 storage 만 분산)
- 분산 SQL DB (CockroachDB / YugabyteDB) 의 selling point: 글로벌 + 강한 일관성
- Spanner = Google 의 **TrueTime** (atomic clock) 기반 — 진짜 external consistency

---

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Instance** | compute (node 또는 PU) |
| **Database** | schema + tables |
| **Node** = 1000 PU (Processing Unit) | 최소 100 PU (~ $65/월) |
| **Region / Multi-region** | regional or multi-region |
| **Interleave** | 부모-자식 row 같은 split (locality) |

---

## 4. 사용

```sql
CREATE TABLE users (
  user_id STRING(36),
  email STRING(320),
  name STRING(100),
) PRIMARY KEY (user_id);

CREATE TABLE orders (
  user_id STRING(36),
  order_id STRING(36),
  amount INT64,
) PRIMARY KEY (user_id, order_id),
INTERLEAVE IN PARENT users ON DELETE CASCADE;
```

→ `INTERLEAVE` 로 부모 row 와 같은 split → JOIN 빠름.

```bash
gcloud spanner instances create myapp \
  --config=regional-asia-northeast3 \
  --processing-units=100

gcloud spanner databases create app --instance=myapp
gcloud spanner databases ddl update app --instance=myapp --ddl="CREATE TABLE ..."
```

---

## 5. 비용

```
Regional: $0.65 / PU·시간 → 100 PU = $65/월
Multi-region: $1.62 / PU·시간 → 100 PU = $162/월
Storage: $0.30 / GB·월 (regional), $0.50 (multi-region)
```

→ 작은 시작도 $65/월. 가격이 진입 장벽.

---

## 6. 사용 시나리오

- 글로벌 OLTP (금융 / 게임 / SaaS)
- 강한 일관성 + 거대 scale
- multi-region active-active
- regulatory + geographic 데이터 분산

부적합:
- 작은 / 중간 OLTP — Cloud SQL 이 쌈
- 분석 — BigQuery
- 단순 — overkill

---

## 7. 함정

- 가격 (PU 단위)
- schema design — interleave 잘못 = JOIN 비싸짐
- SQL dialect 차이 (Postgres dialect 지원)
- read-only transaction 으로 latency ↓

---

## 8. 관련

- [[database]]
- [[cloud-sql]] — 간단 / 작은
- [[../../../database/cassandra/cassandra]] — 비교 NoSQL
