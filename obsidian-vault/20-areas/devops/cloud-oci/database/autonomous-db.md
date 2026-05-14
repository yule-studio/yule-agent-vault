---
title: "OCI Autonomous Database"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:45:00+09:00
tags:
  - oci
  - database
  - autonomous-db
  - oracle
---

# OCI Autonomous Database

**[[database|↑ Database]]** · **[[../cloud-oci|↑↑ OCI]]**

---

## 1. 한 줄

Oracle 의 **self-driving Oracle Database** — 자동 튜닝 / 패치 / 백업 / 보안. OCI 의 시그니처. 2 instance Always Free.

---

## 2. Workload 종류

| 종류 | 의미 |
| --- | --- |
| **ATP** (Autonomous Transaction Processing) | OLTP — 거래 응용 |
| **ADW** (Autonomous Data Warehouse) | OLAP — 분석 |
| **AJD** (Autonomous JSON DB) | document store |
| **APEX** | 응용 개발 |

→ 거의 같은 엔진, 튜닝 / 인덱스 / 스토리지 다름.

---

## 3. Deployment

| Type | 의미 |
| --- | --- |
| **Shared (Serverless)** | 비용 효율 + auto scale + Always Free |
| **Dedicated** | 격리 + Exadata 인프라 |

---

## 4. 사용

```hcl
resource "oci_database_autonomous_database" "adb" {
  compartment_id           = var.compartment_ocid
  db_name                  = "myapp"
  display_name             = "myapp-adb"
  admin_password           = var.adb_password
  cpu_core_count           = 1
  data_storage_size_in_tbs = 1
  db_workload              = "OLTP"                  # 또는 DW
  is_auto_scaling_enabled  = true
  license_model            = "LICENSE_INCLUDED"
  whitelisted_ips          = ["..."]
  is_free_tier             = false
}
```

```bash
oci db autonomous-database create \
  --compartment-id <ocid> \
  --db-name myapp \
  --admin-password '<password>' \
  --cpu-core-count 1 \
  --data-storage-size-in-tbs 1 \
  --db-workload OLTP \
  --is-auto-scaling-enabled true \
  --is-free-tier true                             # Always Free
```

---

## 5. 접속

```bash
# Wallet 다운로드 (mTLS)
oci db autonomous-database generate-wallet \
  --autonomous-database-id <id> --password '<wallet-pwd>' \
  --file wallet.zip

# Python (cx_Oracle / oracledb)
pip install oracledb
```

```python
import oracledb
oracledb.init_oracle_client(config_dir="./wallet")     # 또는 thin mode (wallet X)

conn = oracledb.connect(
    user="admin",
    password="<password>",
    dsn="myapp_high",                                  # tnsnames.ora 의 service
    config_dir="./wallet",
    wallet_location="./wallet",
    wallet_password="<wallet-pwd>",
)
cur = conn.cursor()
cur.execute("SELECT sysdate FROM dual")
print(cur.fetchone())
```

또는 connection string `tcps://...` (mTLS).

---

## 6. Auto Scaling

```
CPU 1 → 자동 1-3x (CPU pressure)
스토리지 1 TB → 자동 확장
```

비용은 사용한 만큼만.

---

## 7. APEX (low-code)

ATP 안에서 APEX 자동 활성 — REST API + UI 개발.

---

## 8. SELECT AI (Generative AI)

```sql
SELECT AI 'sales 가 가장 큰 5 개 region';
```

→ 자연어 → SQL.

---

## 9. 가격

```
OCPU (Shared):    $0.36 / OCPU·시간 (License Included)
Storage:           $0.025 / GB·월
Always Free:      2 ADB × 20 GB
```

→ License Included = Oracle DB 라이선스 포함. BYOL 약 1/3.

---

## 10. 함정

- mTLS wallet 보호 필요
- whitelist IP — 0.0.0.0/0 위험
- auto scale 의 max → 비용 폭증 가능
- backup 무료 (60일 retention)
- clone / refreshable clone — DEV / TEST 환경 강력

---

## 11. 관련

- [[database]]
- [[mysql-heatwave]]
- [[../../../database/oracle/oracle|↗ Oracle DB 이론]]
