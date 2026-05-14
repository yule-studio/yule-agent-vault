---
title: "Azure SQL Database"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:50:00+09:00
tags:
  - azure
  - database
  - sql-server
---

# Azure SQL Database

**[[database|↑ Database]]** · **[[../cloud-azure|↑↑ Azure]]**

---

## 1. 한 줄

매니지드 **SQL Server** — PaaS. Single Database / Elastic Pool / Managed Instance.

---

## 2. Tier (purchasing model)

### 2.1 DTU-based (옛)
DTU = CPU + memory + I/O 의 묶음.

### 2.2 vCore (권장)
| Tier | 의미 |
| --- | --- |
| General Purpose | 균형 |
| Business Critical | 최고 IOPS / HA |
| Hyperscale | 최대 100TB + 빠른 backup |
| Serverless | 자동 pause / scale |

---

## 3. 사용

```bash
az sql server create -g myapp -n myapp-sql -l koreacentral \
  -u sqladmin -p '<password>'

az sql db create -g myapp -s myapp-sql -n app \
  --service-objective GP_Gen5_2 \
  --backup-storage-redundancy Local
```

```hcl
resource "azurerm_mssql_server" "sql" {
  name                         = "myapp-sql"
  resource_group_name          = azurerm_resource_group.main.name
  location                     = "koreacentral"
  version                      = "12.0"
  administrator_login          = "sqladmin"
  administrator_login_password = var.password
  minimum_tls_version          = "1.2"
  azuread_administrator {
    login_username = "admin@example.com"
    object_id      = "..."
  }
}

resource "azurerm_mssql_database" "app" {
  name           = "app"
  server_id      = azurerm_mssql_server.sql.id
  sku_name       = "GP_S_Gen5_2"           # Serverless
  min_capacity   = 0.5
  auto_pause_delay_in_minutes = 60
  max_size_gb    = 50
  geo_backup_enabled = true
}
```

---

## 4. 접속

```python
import pyodbc
conn = pyodbc.connect(
    "Driver={ODBC Driver 18 for SQL Server};"
    "Server=myapp-sql.database.windows.net,1433;"
    "Database=app;"
    "Encrypt=yes;"
    "Authentication=ActiveDirectoryDefault"
)
```

→ Entra ID auth 권장 (password 없이 Managed Identity).

---

## 5. Backup

- automatic backup (7 일 default)
- LTR (Long-Term Retention) — 10 년까지
- PITR (Point-In-Time Recovery)
- geo-restore (cross-region)

---

## 6. 가격

```
GP_Gen5_2 (Provisioned): ~$370/월
GP_S_Gen5 (Serverless): ~$0.5 / vCore·시간 활성 시
Storage:                  $0.13/GB·월
Backup storage:           free up to DB size
```

→ Serverless = 변동 / dev 환경.

---

## 7. 함정

- TDE (encryption) 기본 활성 — 옛 코드 호환 주의
- firewall rule (server level + database level)
- max DTU / vCore 한계 — tier 업그레이드
- BACPAC export = 큰 DB 느림
- Managed Instance = 전체 SQL Server 호환 (옛 응용)

---

## 8. 관련

- [[database]]
- [[../security/key-vault]] — connection string
- [[../../../database/postgresql/postgresql]] — 비교 OSS
