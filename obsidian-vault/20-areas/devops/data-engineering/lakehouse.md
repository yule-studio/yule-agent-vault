---
title: "Lakehouse — Delta / Iceberg / Hudi"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:08:00+09:00
tags: [devops, data-engineering, lakehouse, iceberg, delta]
---

# Lakehouse — Delta / Iceberg / Hudi

**[[data-engineering|↑ data-engineering]]**

---

## 1. 무엇

```
Data Lake (S3):
  + cheap storage
  + scalable
  - no schema enforcement
  - no transaction (ACID)
  - no time travel
  - hard to update (parquet immutable)

Data Warehouse (Snowflake / BigQuery):
  + ACID
  + fast query
  - 비쌈
  - vendor lock-in
  - structured 만

Lakehouse (★ best of both):
  - cheap S3 storage
  - + ACID transaction
  - + time travel
  - + schema evolution
  - + upsert / delete
  - + open format
```

→ "data lake + warehouse" = lakehouse.

---

## 2. table format 3 종

```
parquet 위에 metadata layer 추가:

Apache Iceberg (★ 2024 표준 추세)
  - Netflix 시작
  - vendor-neutral
  - REST catalog (cross-engine)
  - 큰 회사 채택 (AWS / Cloudera / Snowflake / Databricks / 모두)

Delta Lake
  - Databricks 시작
  - 가장 mature
  - Spark + Databricks 최적
  - OSS protocol

Apache Hudi
  - Uber 시작
  - upsert / streaming 강
  - Spark / Flink
```

→ 새 프로젝트 = Iceberg 권장 (interoperability).

---

## 3. 기능 비교

| | Delta | Iceberg | Hudi |
| --- | --- | --- | --- |
| ACID | ✓ | ✓ | ✓ |
| time travel | ✓ | ✓ | ✓ |
| schema evolution | ✓ | ✓ (★) | ✓ |
| partition evolution | △ | ✓ (★) | △ |
| upsert | ✓ | ✓ | ✓ (★) |
| streaming write | ✓ | ✓ | ✓ (★) |
| concurrent write | ✓ | ✓ (★ optimistic) | △ |
| cross-engine | △ | ✓ (★) | △ |
| catalog | Unity (proprietary) | REST (open) | none |
| performance | Spark 최적 | balanced | Spark 최적 |

---

## 4. Iceberg 의 구조

```
warehouse/
└── db/
    └── orders/
        ├── data/
        │   ├── ... parquet files
        ├── metadata/
        │   ├── v1.metadata.json
        │   ├── v2.metadata.json
        │   ├── snap-123.avro       (snapshot)
        │   └── manifest-list-456.avro

metadata 가 어느 file 이 어느 snapshot 에 있는지 추적.
→ atomic commit + time travel.
```

---

## 5. Iceberg + Spark

```python
# Spark + Iceberg 사용
spark = (SparkSession.builder
    .config("spark.sql.extensions", "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions")
    .config("spark.sql.catalog.local", "org.apache.iceberg.spark.SparkCatalog")
    .config("spark.sql.catalog.local.type", "hadoop")
    .config("spark.sql.catalog.local.warehouse", "s3://my-warehouse")
    .getOrCreate())

# create
spark.sql("""
    CREATE TABLE local.db.orders (
        id BIGINT,
        user_id BIGINT,
        amount DECIMAL(10,2),
        created_at TIMESTAMP
    )
    USING iceberg
    PARTITIONED BY (days(created_at))
""")

# insert
spark.sql("INSERT INTO local.db.orders VALUES (1, 100, 50.00, current_timestamp())")

# update (★)
spark.sql("UPDATE local.db.orders SET amount = 60.00 WHERE id = 1")

# delete
spark.sql("DELETE FROM local.db.orders WHERE id = 1")

# merge (upsert)
spark.sql("""
    MERGE INTO local.db.orders t
    USING source s
    ON t.id = s.id
    WHEN MATCHED THEN UPDATE SET *
    WHEN NOT MATCHED THEN INSERT *
""")

# time travel
spark.read.option("snapshot-id", "123456789").table("local.db.orders")
spark.read.option("as-of-timestamp", "2026-05-15 10:00:00").table("local.db.orders")

# 옛 snapshot 정리
spark.sql("CALL local.system.expire_snapshots('local.db.orders', TIMESTAMP '2026-04-01 00:00:00')")
```

---

## 6. Delta Lake (Databricks 친화)

```python
# Spark + Delta
from delta.tables import *

df.write.format("delta").save("/data/orders")

# table 등록
spark.sql("CREATE TABLE orders USING DELTA LOCATION '/data/orders'")

# upsert
delta_table = DeltaTable.forPath(spark, "/data/orders")
delta_table.alias("t").merge(
    source.alias("s"),
    "t.id = s.id"
).whenMatchedUpdate(set={"amount": "s.amount"}) \
 .whenNotMatchedInsertAll() \
 .execute()

# time travel
spark.read.option("versionAsOf", 5).format("delta").load("/data/orders")
spark.read.option("timestampAsOf", "2026-05-15").format("delta").load("/data/orders")

# OPTIMIZE (compact small files)
spark.sql("OPTIMIZE orders WHERE created_at >= '2026-01-01'")

# VACUUM (옛 file 삭제)
spark.sql("VACUUM orders RETAIN 168 HOURS")  # 7일
```

---

## 7. catalog (★ 중요)

```
metadata 추적 필요:

Hive Metastore (전통):
  - Hadoop 시작
  - 가장 흔함

AWS Glue Catalog:
  - AWS 의 Hive Metastore
  - Athena / EMR / Redshift Spectrum 자동

Unity Catalog (Databricks):
  - lineage / RBAC 포함
  - 강력 but proprietary

Iceberg REST Catalog (★ open):
  - cross-engine
  - Tabular.io / Snowflake / AWS / Apache
  - 2024 표준 추세

Nessie (Project Nessie):
  - git-like (branch / tag)
  - Iceberg / Delta 지원
```

---

## 8. compaction (★)

```
streaming write / 작은 file 누적:
  - 한 partition 에 수천 작은 file
  - read 느림

compaction:
  - 작은 file → 큰 file 묶음
  - 정기 schedule
  - 또는 자동 (Hudi)

Iceberg:
  CALL system.rewrite_data_files('orders')

Delta:
  OPTIMIZE orders
```

→ data lake 의 hidden 운영 부담.

---

## 9. partition strategy

```
시간 기반 (가장 흔함):
  PARTITIONED BY (days(created_at))
  → daily partition. 7-30일 단위 query 적합

multi-level:
  PARTITIONED BY (region, days(created_at))
  → region 별 + 시간

bucketed (large dim):
  PARTITIONED BY (bucket(16, user_id))
  → join 시 효율

→ Iceberg 의 partition evolution = 미래 변경 OK (재partition 안 해도).
```

---

## 10. streaming (★)

```python
# Spark Structured Streaming + Iceberg
df = (spark.readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "kafka:9092")
    .option("subscribe", "events")
    .load())

# transform
events = df.selectExpr("CAST(value AS STRING)") \
    .select(from_json("value", schema).alias("e")) \
    .select("e.*")

# write to Iceberg (streaming)
query = (events.writeStream
    .format("iceberg")
    .option("checkpointLocation", "/checkpoint")
    .outputMode("append")
    .trigger(processingTime="1 minute")
    .toTable("local.db.events"))

query.awaitTermination()
```

→ "kafka → Iceberg" 1분 마다. micro-batch.

---

## 11. cost 절감

```
- file format = parquet + zstd / snappy
- partition 적절
- compaction (작은 file 합침)
- statistics (Iceberg 의 manifest)
- column pruning (필요 column 만)
- predicate pushdown
- materialized view (자주 쓰는 aggregate)
- partition lifecycle (옛 → Glacier)
```

---

## 12. governance (★)

```
- schema enforce + evolution
- access control (column-level)
- audit log (누가 query)
- lineage (table A → table B)
- PII tagging
- data quality (Great Expectations)
- contract (producer commit)
```

도구:
- Unity Catalog
- AWS Lake Formation
- DataHub
- Open Lineage
- Apache Atlas

---

## 13. read engine

```
같은 lakehouse data 를 여러 engine 으로:

Spark:        batch + streaming
Trino/Presto: SQL ad-hoc
Athena:       managed (AWS)
BigQuery:     external table (GCP)
Snowflake:    external Iceberg table
DuckDB:       local query
Flink:        streaming
Dremio:       BI engine
```

→ Iceberg 의 강점 = cross-engine.

---

## 14. 함정

1. **compaction 안 함** — 작은 file 폭주 → query 느림.
2. **catalog 잘못 선택** — Unity = lock-in.
3. **table version 무한 누적** — `VACUUM` / `expire_snapshots` 정기.
4. **schema evolution 무지** — backward compatible 만.
5. **partition 너무 fine** — Postgres 의 partition 처럼 N 폭주.
6. **streaming + non-idempotent transform** — duplicate.
7. **statistics 갱신 안 함** — query planner 잘못.
8. **PII 평문** — column-level encryption.

---

## 15. 관련

- [[data-engineering|↑ data-engineering]]
- [[spark]]
- [[../finops/architecture-optimization|↗ S3 cost]]
