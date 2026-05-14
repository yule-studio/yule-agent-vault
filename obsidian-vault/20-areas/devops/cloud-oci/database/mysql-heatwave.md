---
title: "OCI MySQL HeatWave"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:50:00+09:00
tags:
  - oci
  - database
  - mysql
  - heatwave
---

# OCI MySQL HeatWave

**[[database|↑ Database]]** · **[[../cloud-oci|↑↑ OCI]]**

---

## 1. 한 줄

**MySQL + in-memory OLAP accelerator** — 같은 DB 에서 transactional + analytical (HTAP). RDS MySQL 보다 OLAP 100x+ 흔히.

---

## 2. 사용

```hcl
resource "oci_mysql_mysql_db_system" "mysql" {
  compartment_id      = var.compartment_ocid
  shape_name          = "MySQL.HeatWave.VM.Standard.E3"
  availability_domain = data.oci_identity_availability_domain.ad.name
  subnet_id           = oci_core_subnet.db.id
  admin_username      = "admin"
  admin_password      = var.mysql_password
  data_storage_size_in_gb = 50

  is_highly_available = true                # HA

  backup_policy {
    is_enabled = true
    retention_in_days = 30
  }
}

# HeatWave cluster 활성 (option)
resource "oci_mysql_heat_wave_cluster" "hw" {
  db_system_id    = oci_mysql_mysql_db_system.mysql.id
  cluster_size    = 2
  shape_name      = "MySQL.HeatWave.VM.Standard.E3"
}
```

---

## 3. HeatWave 사용

```sql
-- 1. HeatWave 에 테이블 load
ALTER TABLE orders SECONDARY_ENGINE = RAPID;
ALTER TABLE orders SECONDARY_LOAD;

-- 2. 그냥 query
SELECT region, SUM(amount) FROM orders GROUP BY region;
-- → optimizer 가 자동으로 HeatWave 사용
```

→ 응용 코드 변경 없음.

---

## 4. Lakehouse

S3-호환 / OCI Object Storage 의 CSV / Parquet / Avro / JSON 을 직접 query — 외부 테이블.

---

## 5. AutoML / Vector / GenAI

HeatWave 안에 ML / Vector / LLM 통합.

```sql
CALL sys.ML_TRAIN('sales','quantity', JSON_OBJECT('task','regression'), @model);
```

---

## 6. 가격

```
VM.Standard.E3 (1 OCPU/16 GB): $0.0521 / 시간
HeatWave node: + $0.04 / OCPU·시간
Storage: $0.025 / GB·월
```

→ RDS MySQL 비슷한 가격대 + HeatWave 추가 가능.

---

## 7. 함정

- HeatWave 는 read-only (transactional 은 MySQL primary)
- HeatWave load 메모리 충분해야
- backup = Object Storage
- HA = 3-node (RAFT)

---

## 8. 관련

- [[database]]
- [[autonomous-db]]
- [[../../../database/mysql/mysql|↗ MySQL]]
