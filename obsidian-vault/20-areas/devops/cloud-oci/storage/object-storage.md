---
title: "OCI Object Storage"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:30:00+09:00
tags:
  - oci
  - storage
  - object-storage
  - s3-compatible
---

# OCI Object Storage

**[[storage|↑ Storage]]** · **[[../cloud-oci|↑↑ OCI]]**

---

## 1. 한 줄

OCI 의 object storage — **S3-compatible API** (boto3 그대로). 11 nines durability. 20 GB Always Free.

---

## 2. Tier

| Tier | 의미 |
| --- | --- |
| **Standard** | 빈번 access |
| **Infrequent Access** | 월별 |
| **Archive** | 분기 + 시간 단위 retrieval |

---

## 3. 사용

```bash
oci os bucket create -ns <namespace> --name myapp-data \
  --compartment-id <ocid> \
  --storage-tier Standard \
  --versioning Enabled

oci os object put -ns <ns> -bn myapp-data --file ./data.json --name uploads/data.json
oci os object get -ns <ns> -bn myapp-data --name uploads/data.json --file out.json
oci os object list -ns <ns> -bn myapp-data
```

```hcl
data "oci_objectstorage_namespace" "ns" {
  compartment_id = var.tenancy_ocid
}

resource "oci_objectstorage_bucket" "data" {
  compartment_id = var.compartment_ocid
  namespace      = data.oci_objectstorage_namespace.ns.namespace
  name           = "myapp-data"
  access_type    = "NoPublicAccess"
  storage_tier   = "Standard"
  versioning     = "Enabled"
  kms_key_id     = oci_kms_key.bucket.id
}
```

---

## 4. S3-compatible

```python
import boto3
s3 = boto3.client(
    "s3",
    region_name="ap-seoul-1",
    endpoint_url="https://<namespace>.compat.objectstorage.ap-seoul-1.oraclecloud.com",
    aws_access_key_id="...",          # OCI Customer Secret Key
    aws_secret_access_key="..."
)
s3.put_object(Bucket="myapp-data", Key="data.json", Body=b"{}")
```

→ 기존 S3 코드 재사용 가능.

---

## 5. Pre-Authenticated Request (PAR)

```bash
oci os preauth-request create -ns <ns> -bn myapp-data \
  --name upload-link \
  --access-type ObjectReadWrite \
  --time-expires '2026-06-01T00:00:00Z'
```

→ S3 presigned URL 동등.

---

## 6. 이벤트 → Function

Object Storage 의 이벤트 (생성 / 삭제) → Events Service → Function trigger.

---

## 7. 비용

```
Standard:       $0.0255 / GB·월
Infrequent:     $0.01 / GB·월
Archive:        $0.0026 / GB·월

Operations: Standard $0.0034 / 10K (write)
Egress:    10 TB/월 무료 (OCI 의 평생 무료)
```

→ AWS / Azure 보다 outbound 대폭 저렴.

---

## 8. 함정

- bucket name = compartment 안 unique (글로벌 unique X — S3 와 다름)
- namespace = tenancy 별 1 개
- public access disable (default)
- lifecycle = tier 자동 전환 + 만료 삭제
- 200 GB Always Free (object + block 합산)

---

## 9. 관련

- [[storage]]
- [[block-volume]]
- [[../security/vault]] — encryption key
