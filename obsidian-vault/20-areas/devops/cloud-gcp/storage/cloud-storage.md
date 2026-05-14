---
title: "GCP Cloud Storage (GCS)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:30:00+09:00
tags:
  - gcp
  - storage
  - gcs
---

# GCP Cloud Storage (GCS)

**[[storage|↑ Storage]]** · **[[../cloud-gcp|↑↑ GCP]]**

---

## 1. 한 줄

GCP 의 **object storage** — AWS S3 동등. 글로벌 + region / multi-region / dual-region 옵션.

---

## 2. Storage Class

| Class | $/GB·월 | min 보관 | 용도 |
| --- | --- | --- | --- |
| Standard | $0.020 | - | 핫 |
| Nearline | $0.010 | 30일 | 월별 |
| Coldline | $0.004 | 90일 | 분기 |
| Archive | $0.0012 | 365일 | 장기 |

→ Lifecycle 로 자동 전환.

---

## 3. 설치 / 사용

### 3.1 CLI (gsutil / gcloud)

```bash
# bucket
gsutil mb -l asia-northeast3 gs://my-bucket
gcloud storage buckets create gs://my-bucket --location=asia-northeast3

# 업로드
gsutil cp file.txt gs://my-bucket/
gcloud storage cp file.txt gs://my-bucket/
gcloud storage rsync ./local gs://my-bucket/dir

# 다운로드
gcloud storage cp gs://my-bucket/file.txt .

# 목록
gcloud storage ls gs://my-bucket/dir/

# 삭제
gcloud storage rm gs://my-bucket/file.txt

# Signed URL
gcloud storage sign-url --duration=1h gs://my-bucket/file.txt
```

### 3.2 SDK (Python)

```python
from google.cloud import storage
client = storage.Client()
bucket = client.bucket("my-bucket")
bucket.blob("file.txt").upload_from_filename("local.txt")
bucket.blob("file.txt").download_to_filename("out.txt")

# Signed URL
url = bucket.blob("file.txt").generate_signed_url(version="v4", expiration=3600, method="PUT")
```

### 3.3 Terraform

```hcl
resource "google_storage_bucket" "data" {
  name                        = "my-data-bucket"
  location                    = "ASIA-NORTHEAST3"
  uniform_bucket_level_access = true            # IAM-only (권장)
  storage_class               = "STANDARD"

  versioning { enabled = true }

  lifecycle_rule {
    condition { age = 30 }
    action    { type = "SetStorageClass"; storage_class = "NEARLINE" }
  }
  lifecycle_rule {
    condition { age = 365 }
    action    { type = "Delete" }
  }

  encryption { default_kms_key_name = google_kms_crypto_key.gcs.id }
}
```

---

## 4. 보안 — public 차단

**Uniform Bucket-Level Access (UBLA)** + **Public Access Prevention** 권장 — IAM 으로만 access, ACL 사용 X.

```bash
gcloud storage buckets update gs://my-bucket --uniform-bucket-level-access
gcloud storage buckets update gs://my-bucket --public-access-prevention=enforced
```

---

## 5. 이벤트 통합

```bash
# Pub/Sub 통지
gcloud storage buckets notifications create gs://my-bucket \
  --topic=projects/PROJECT/topics/file-events \
  --event-types=OBJECT_FINALIZE

# 또는 EventArc → Cloud Run / Functions
```

---

## 6. 정적 웹사이트

```bash
gcloud storage buckets update gs://my-site \
  --web-main-page-suffix=index.html \
  --web-error-page=404.html
```

CloudFront 대신 Cloud CDN 결합.

---

## 7. 비용 (Seoul)

```
Storage   $0.020 / GB·월 (Standard)
Class A operation (write/list) $0.005 / 1K
Class B operation (read)         $0.0004 / 1K
Egress    region 내부 = 무료
          같은 region 다른 zone = 무료
          다른 region 한국 → 미국 = $0.12/GB
Lifecycle 자동 전환 = 무료
```

---

## 8. 함정

### 8.1 일부 region 의 lock
주의: bucket 의 location 변경 X. 처음 신중.

### 8.2 versioning + delete
delete marker 만 — storage 비용 유지. Lifecycle 로 옛 버전 만료.

### 8.3 ACL vs IAM
UBLA 활성 권장 — ACL 헷갈림.

### 8.4 multi-region 비용
us / eu / asia 등 multi-region — 비싸지만 자동 replication.

### 8.5 egress 인터넷
$0.12/GB (한국 → 글로벌). Cloud CDN 사용 시 절약.

---

## 9. 사용 시나리오

- 정적 사이트 / 이미지 / 비디오
- 백업 / archive
- data lake (BigQuery 와 통합)
- container image (Artifact Registry 의 backing)
- ML training data

---

## 10. 학습 자료

- GCP Cloud Storage docs
- **GCS Best Practices**

---

## 11. 관련

- [[storage]] — Storage hub
- [[persistent-disk]] — block
- [[../../../database/elasticsearch/elasticsearch]] — snapshot
- [[../security/cloud-kms]] — 암호화
