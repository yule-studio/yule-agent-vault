---
title: "GCP Database (Hub)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:35:00+09:00
tags:
  - gcp
  - database
  - hub
---

# GCP Database (Hub)

**[[../cloud-gcp|↑ GCP]]**

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **Cloud SQL** | [[cloud-sql]] | 매니지드 Postgres / MySQL / SQL Server |
| **Spanner** | [[spanner]] | 글로벌 분산 RDB (강한 일관성) |
| **Firestore** | [[firestore]] | Document NoSQL |
| **BigQuery** | [[bigquery]] | DW / 분석 |

추가:
- **AlloyDB** — Postgres 호환 + 분산 (Aurora 동등)
- **Bigtable** — Cassandra 동등 NoSQL (HBase API)
- **Memorystore** — Redis / Memcached

---

## 선택

| 시나리오 | 추천 |
| --- | --- |
| OLTP / 일반 | Cloud SQL Postgres |
| 글로벌 / 강한 일관성 | Spanner |
| Document / 모바일 / 실시간 | Firestore |
| 분석 / DW | BigQuery |
| 거대 KV (Cassandra) | Bigtable |
| 캐시 | Memorystore Redis |

---

## 관련

- [[../cloud-gcp|↑ GCP]]
- [[../../../database/database]]
