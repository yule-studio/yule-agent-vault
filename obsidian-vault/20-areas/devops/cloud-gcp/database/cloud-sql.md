---
title: "GCP Cloud SQL"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:40:00+09:00
tags:
  - gcp
  - database
  - cloud-sql
---

# GCP Cloud SQL

**[[database|↑ Database]]** · **[[../cloud-gcp|↑↑ GCP]]**

---

## 1. 한 줄

매니지드 RDB — Postgres / MySQL / SQL Server. AWS RDS 동등.

---

## 2. 설치

```bash
gcloud sql instances create myapp-pg \
  --database-version=POSTGRES_16 \
  --tier=db-custom-2-7680 \
  --region=asia-northeast3 \
  --availability-type=REGIONAL \
  --storage-type=SSD \
  --storage-size=100 \
  --storage-auto-increase \
  --backup-start-time=18:00 \
  --enable-bin-log

gcloud sql databases create app --instance=myapp-pg
gcloud sql users create app --instance=myapp-pg --password=...
```

```hcl
resource "google_sql_database_instance" "myapp" {
  name             = "myapp-pg"
  database_version = "POSTGRES_16"
  region           = "asia-northeast3"

  settings {
    tier              = "db-custom-2-7680"          # 2 vCPU, 7.5 GB
    availability_type = "REGIONAL"                   # HA
    disk_size         = 100
    disk_type         = "PD_SSD"
    disk_autoresize   = true

    backup_configuration {
      enabled                        = true
      point_in_time_recovery_enabled = true
      start_time                     = "18:00"
      backup_retention_settings {
        retained_backups = 14
      }
    }

    ip_configuration {
      ipv4_enabled    = false
      private_network = google_compute_network.vpc.id
    }

    database_flags {
      name  = "max_connections"
      value = "200"
    }

    insights_config {
      query_insights_enabled  = true
      record_application_tags = true
      record_client_address   = true
    }
  }

  deletion_protection = true
}
```

---

## 3. 접속 — Cloud SQL Auth Proxy (권장)

```bash
# proxy 다운로드
curl -o cloud-sql-proxy https://dl.google.com/cloudsql/cloud-sql-proxy.linux.amd64
chmod +x cloud-sql-proxy

./cloud-sql-proxy PROJECT:asia-northeast3:myapp-pg &

# localhost:5432 → Cloud SQL (TLS + IAM 자동)
psql "host=127.0.0.1 user=app dbname=app"
```

→ private IP 직접 보다 안전 + IAM 인증.

### 3.1 IAM Authentication

```sql
GRANT CONNECT ON DATABASE app TO "myuser@PROJECT.iam";
```

```bash
# token (1시간)
TOKEN=$(gcloud auth print-access-token)
PGPASSWORD=$TOKEN psql "host=127.0.0.1 user=myuser@PROJECT.iam dbname=app"
```

→ password 보관 X.

### 3.2 Cloud Run / GKE 통합

Cloud Run:
```hcl
template {
  containers { ... }
  vpc_access { connector = ... }
}
# 환경변수 INSTANCE_CONNECTION_NAME = PROJECT:REGION:INSTANCE
```

---

## 4. HA / 백업

- **Regional** = primary + standby (synchronous, 다른 zone)
- automatic backup + PITR
- read replica (cross-region 가능)
- failover ~ 60-120 초

---

## 5. 비용

```
db-custom-2-7680 single zone:  ~ $100/월
                regional:        ~ $200/월 (2배)
Storage SSD:                     $0.187/GB·월
Backup:                          첫 100% 무료, 그 후 $0.08/GB
```

---

## 6. 함정

- maintenance window = 짧은 다운타임
- max_connections 제한 — connection pool (PgBouncer)
- private network 권장 — public ip = 위험
- minor 자동 upgrade 권장 (회귀 검증 후)
- deletion_protection 끄지 말 것

---

## 7. 관련

- [[database]]
- [[../security/secret-manager]]
- [[../../../database/postgresql/postgresql|↗ PG 자체]]
