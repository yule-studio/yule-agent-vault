---
title: "Data Engineering — Kafka / ETL / Airflow / Spark ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:02:00+09:00
tags: [area, devops, data-engineering]
---

# Data Engineering — Kafka / ETL / Airflow / Spark ★

**[[../devops|↑ devops]]**

---

## 1. 영역

```
1. ingestion           — Kafka / Kinesis / CDC
2. storage             — data lake (S3) / lakehouse (Delta/Iceberg)
3. transformation      — Spark / dbt / Flink
4. orchestration       — Airflow / Dagster / Prefect
5. analytics           — Trino / Athena / BigQuery
6. governance          — schema / lineage / quality
7. real-time           — streaming (Flink / Spark Streaming)
8. ML feature          — Feature Store (Feast / Tecton)
```

→ DevOps 시니어 = data 팀의 infra / pipeline 협업.

---

## 2. 하위 영역

- [[data-pipeline]] — ETL vs ELT
- [[kafka-deep]] — Kafka 운영 deep
- [[airflow]] — DAG / operator / sensor
- [[dbt]] — SQL transformation
- [[spark]] — batch / streaming
- [[lakehouse]] — Delta / Iceberg / Hudi
- [[data-quality]] — Great Expectations / Soda
- [[data-governance]] — schema / lineage / catalog
- [[real-time-streaming]] — Flink / Kafka Streams
- [[cdc-debezium]] — change data capture
- [[pitfalls]]

---

## 3. 표준 stack

```
ingestion:
  Kafka (real-time)
  Kafka Connect / Debezium (CDC)
  Airbyte (3rd-party SaaS data)
  Fivetran (SaaS)

storage:
  S3 / GCS / ABFS
  Iceberg / Delta Lake (table format)

processing:
  Spark (batch + structured streaming)
  Flink (real-time)
  dbt (SQL transformation)
  Trino / Presto (query)

orchestration:
  Airflow (popular)
  Dagster (modern)
  Prefect

analytics / BI:
  Looker / Tableau / Superset
  BigQuery / Snowflake / Redshift (warehouse)

governance:
  Unity Catalog (Databricks)
  AWS Lake Formation
  DataHub (LinkedIn)
  Apache Atlas
  Amundsen

quality:
  Great Expectations
  Soda
  Monte Carlo

ML feature:
  Feast / Tecton / Hopsworks
```

---

## 4. ETL vs ELT (★)

```
ETL (전통):
  Extract → Transform → Load
  → 작은 warehouse 시대
  → transform 이 비싸 / 느림

ELT (★ modern):
  Extract → Load → Transform
  → S3 / lakehouse 에 raw load
  → SQL (dbt) 으로 transform
  → 큰 compute (Spark / Snowflake) 활용
  → 유연 (raw 보존, schema 변경 쉬움)
```

→ **modern = ELT**. data warehouse / lakehouse 시대.

---

## 5. lakehouse architecture (★)

```
[source]
  ↓
[ingest (Kafka / batch)]
  ↓
[raw / bronze]   — S3 + Iceberg/Delta
  ↓ clean
[silver]          — schema 정리 / dedup
  ↓ business logic
[gold]            — aggregated / report-ready
  ↓
[analytics (Trino, BI)]
[ML (training)]
```

→ medallion architecture (Databricks 발표).

---

## 6. data 의 책임 분담

```
Software Engineer (DevOps):
  - Kafka cluster 운영
  - Airflow / Spark cluster
  - storage (S3 lifecycle)
  - access (IAM / RBAC)
  - cost monitoring

Data Engineer:
  - pipeline (DAG / dbt model)
  - data quality
  - schema evolution
  - data lineage

Data Analyst:
  - SQL query
  - BI dashboard
  - business KPI

ML Engineer:
  - feature pipeline
  - model training
  - serving

DevOps + DE = infra + pipeline 협업
```

---

## 7. real-time vs batch

```
batch:
  - 매 hour / day
  - 큰 volume, latency 무관
  - Spark / dbt
  - 단순 / cheap

real-time:
  - sub-second
  - 작은 batch
  - Flink / Kafka Streams
  - 복잡 / 비쌈

→ "정말 real-time 필요한가?" 검토.
   대부분 micro-batch (1min) 충분.
```

---

## 8. 시니어 결정 단골

```
"data 폭주, S3 cost ↑"
  → lifecycle (Standard → IA → Glacier)
  → parquet + compression
  → partition (date)
  → 옛 data archive

"Spark job 느림"
  → adaptive query execution (AQE)
  → broadcast join (작은 table)
  → repartition
  → broadcast variable
  → vector UDF

"Airflow DAG 너무 많음"
  → sub-DAG / TaskGroup
  → dynamic task (TaskFlow)
  → Celery executor → Kubernetes executor

"data quality 사고"
  → Great Expectations / Soda
  → schema enforce
  → contract test
  → upstream notify
```

---

## 9. observability for data

```
일반 service monitoring 과 다름:

- pipeline 의 success / fail
- 처리 row 수
- latency (end-to-end)
- backlog
- schema 변경
- data quality (null %, range, ...)
- cost per DAG / job

도구:
  - Airflow UI
  - Great Expectations
  - Monte Carlo
  - DataDog Data Streams Monitoring
  - Datafold
```

---

## 10. modern 추세 (2024-25)

```
- lakehouse (Iceberg/Delta) — 표준화
- streaming-first (Flink + Iceberg)
- declarative (dbt + SQL)
- data contracts (producer 의 schema guarantee)
- data mesh (decentralized)
- data products (각 domain 의 product)
- LLM 의 data 필요 (RAG / fine-tune)
- vector DB (mlops 와 통합)
- Unity Catalog / Iceberg REST catalog (governance)
- compute-storage 분리 (cheap object store + flexible compute)
```

---

## 11. 학습 순서

1. Day 1: [[data-pipeline]] (ETL vs ELT) + [[kafka-deep]]
2. Day 2: [[airflow]] DAG 작성
3. Day 3: [[spark]] (batch + structured streaming)
4. Day 4: [[lakehouse]] (Iceberg / Delta)
5. Day 5: [[data-quality]] + [[cdc-debezium]]

---

## 12. 관련

- [[../devops|↑ devops]]
- [[../distributed-systems/messaging-patterns|↗ Kafka]]
- [[../mlops/mlops|↗ MLOps]]
- [[../monitoring/monitoring|↗ monitoring]]
