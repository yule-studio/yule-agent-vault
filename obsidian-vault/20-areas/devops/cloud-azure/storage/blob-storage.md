---
title: "Azure Blob Storage"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:35:00+09:00
tags:
  - azure
  - storage
  - blob
---

# Azure Blob Storage

**[[storage|↑ Storage]]** · **[[../cloud-azure|↑↑ Azure]]**

---

## 1. 한 줄

Azure 의 object storage — AWS S3 / GCS 동등. 사실상 무제한 + 99.999999999% durability.

---

## 2. 계층

```
Storage Account
  └── Container (= S3 bucket)
        └── Blob (= object)
            - Block blob (일반 파일)
            - Append blob (log)
            - Page blob (VHD)
```

---

## 3. Access Tier

| Tier | $/GB·월 | 의미 |
| --- | --- | --- |
| **Hot** | $0.0184 | 빈번 access |
| **Cool** | $0.01 | 월별 (30+ 일 보관) |
| **Cold** | $0.0036 | 분기 (90+ 일) |
| **Archive** | $0.00099 | 1+ 년 — 복원 분 단위 |

→ Lifecycle policy 자동 전환.

---

## 4. 설치 / 사용

```bash
az storage account create \
  --resource-group myapp \
  --name myappstorage \
  --location koreacentral \
  --sku Standard_LRS \
  --kind StorageV2 \
  --access-tier Hot \
  --https-only true \
  --min-tls-version TLS1_2

az storage container create --account-name myappstorage --name data

az storage blob upload --account-name myappstorage --container-name data --file local.txt --name file.txt
az storage blob download --account-name myappstorage --container-name data --name file.txt --file out.txt
az storage blob list --account-name myappstorage --container-name data
```

```hcl
resource "azurerm_storage_account" "main" {
  name                     = "myappstorage"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = "koreacentral"
  account_tier             = "Standard"
  account_replication_type = "LRS"            # LRS / ZRS / GRS / RA-GRS
  account_kind             = "StorageV2"
  access_tier              = "Hot"
  https_traffic_only_enabled = true
  min_tls_version          = "TLS1_2"

  blob_properties {
    versioning_enabled = true
    delete_retention_policy { days = 30 }
  }
}

resource "azurerm_storage_container" "data" {
  name                  = "data"
  storage_account_name  = azurerm_storage_account.main.name
  container_access_type = "private"
}
```

---

## 5. Replication

| Type | 의미 |
| --- | --- |
| **LRS** | 1 region 안 3 copy |
| **ZRS** | 3 AZ |
| **GRS** | 2 region (primary + secondary) |
| **RA-GRS** | GRS + read-access secondary |
| **GZRS** | ZRS + GRS |

운영 = 보통 ZRS / GRS.

---

## 6. SDK

```python
from azure.storage.blob import BlobServiceClient
client = BlobServiceClient(account_url="https://myappstorage.blob.core.windows.net", credential=token)
blob = client.get_container_client("data").get_blob_client("file.txt")
blob.upload_blob(b"hello", overwrite=True)
data = blob.download_blob().readall()
```

→ Managed Identity 자동 token (key 없이).

---

## 7. SAS Token (시한 URL)

```python
from azure.storage.blob import generate_blob_sas, BlobSasPermissions
from datetime import datetime, timedelta

sas = generate_blob_sas(
    account_name="myappstorage",
    container_name="data",
    blob_name="file.txt",
    permission=BlobSasPermissions(read=True),
    expiry=datetime.utcnow() + timedelta(hours=1),
    account_key="..."
)
url = f"https://myappstorage.blob.core.windows.net/data/file.txt?{sas}"
```

→ 응용 server 가 fronted 한 SAS URL 발급 → 클라이언트 직접 업/다운.

---

## 8. 보안

- **Public access** disable (account level)
- **Network access** — private endpoint / IP rules
- **Encryption at rest** — Microsoft-managed / customer-managed (CMK in Key Vault)
- **Immutable storage** — WORM
- **Soft delete** — 휴지통

---

## 9. 이벤트 → Function / Event Grid

```bash
az eventgrid event-subscription create \
  --source-resource-id <storage-account-id> \
  --name on-upload \
  --endpoint <function-url> \
  --included-event-types Microsoft.Storage.BlobCreated
```

---

## 10. 비용

```
Storage Hot LRS:    $0.0184 / GB·월
Operations:          $0.0043 / 10K (write), $0.00034 (read)
Data Transfer Out:  $0.087 / GB (Asia)
GRS premium:        2-3x
```

---

## 11. 함정

- account name = 글로벌 unique (전세계 1 개)
- public blob = leak 위험 — disable
- versioning + delete = retention cost
- archive tier = read 시 rehydrate (분-시간) + 비용
- ADLS Gen2 (`is_hns_enabled`) — Data Lake / Hadoop 호환 (별도 모드)

---

## 12. 관련

- [[storage]]
- [[files]]
- [[../security/key-vault]]
- [[../../../database/elasticsearch/elasticsearch]]
