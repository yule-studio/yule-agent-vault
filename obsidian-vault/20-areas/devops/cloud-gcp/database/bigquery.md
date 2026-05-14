---
title: "GCP BigQuery"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:55:00+09:00
tags:
  - gcp
  - database
  - bigquery
  - dw
  - analytics
---

# GCP BigQuery

**[[database|↑ Database]]** · **[[../cloud-gcp|↑↑ GCP]]**

---

## 1. 한 줄

**serverless DW** — 페타바이트 SQL. 거대 분석의 사실상 표준 (vs Snowflake / Redshift).

---

## 2. 왜

- 1 TB query = 수초
- 인프라 관리 X
- BI / data lake / ML 통합
- standard SQL

대안:
- **Snowflake** — multi-cloud
- **Redshift** — AWS
- **ClickHouse** — self-managed / 더 싸지만 운영
- **Databricks** — Spark / Delta

---

## 3. 구조

```
Project
  └── Dataset
        └── Table
              └── partition / cluster
```

- **Dataset** = schema 비슷
- **Table** = native / external (GCS / Sheet)
- **Partition** = 일자 / 정수
- **Cluster** = sort key

---

## 4. 첫 query

```sql
SELECT name, COUNT(*) AS cnt
FROM `project.dataset.users`
WHERE country = 'KR'
  AND _PARTITIONTIME BETWEEN '2026-05-01' AND '2026-05-14'
GROUP BY name
ORDER BY cnt DESC
LIMIT 100;
```

```bash
bq query --use_legacy_sql=false 'SELECT ...'
bq load --source_format=PARQUET ds.tbl gs://bucket/file.parquet
bq extract ds.tbl gs://bucket/out.csv
```

---

## 5. 가격 모델

| 모델 | 의미 |
| --- | --- |
| **On-demand** | $6.25 / TB scanned |
| **Capacity (slots)** | reserved compute — 큰 / 일정 부하 |
| **Storage** | $0.020 / GB·월 (active) / $0.010 (long-term) |
| **Streaming** | $0.05 / GB |

→ scan 양 통제 = 비용 통제. partition / cluster 필수.

---

## 6. 데이터 적재

```bash
# Cloud Storage 에서
bq load --source_format=PARQUET ds.tbl gs://bucket/data.parquet

# stream
bq insert ...

# Federated (외부) — GCS / Sheet / Bigtable 직접 query
bq mk --table --external_table_definition=... ds.external_tbl
```

dbt / Dataform / Cloud Composer 가 일반 ETL 도구.

---

## 7. ML 통합

```sql
CREATE OR REPLACE MODEL ds.model
OPTIONS(model_type='linear_reg') AS
SELECT * FROM ds.training;

SELECT * FROM ML.PREDICT(MODEL ds.model, TABLE ds.input);
```

→ SQL 만으로 ML.

---

## 8. 비용 절약

- **partition + cluster** — scan 줄임
- **column 명시** — `SELECT *` 회피
- **dry run** — `bq query --dry_run` 으로 미리 비용 확인
- **materialized view**
- **slot reservation** — 큰 부하

---

## 9. 사용 시나리오

- 회사 DW
- log 분석 (GA4 → BigQuery)
- IoT analytics
- ML pipeline
- 외부 BI (Looker / Tableau / Metabase)

부적합:
- OLTP (Cloud SQL / Spanner)
- 실시간 update
- 작은 record (DocDB)

---

## 10. 함정

- `SELECT *` 비용 폭증
- partition 누락 → 모든 데이터 scan
- streaming insert 비용
- export quota
- region binding

---

## 11. 관련

- [[database]]
- [[../../../database/postgresql/postgresql]] — OLTP 비교
- [[../../../database/elasticsearch/elasticsearch]] — 검색 비교
