---
title: "GCP Cloud CDN"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:15:00+09:00
tags:
  - gcp
  - network
  - cdn
---

# GCP Cloud CDN

**[[network|↑ Network]]** · **[[../cloud-gcp|↑↑ GCP]]**

---

## 1. 한 줄

GCP 의 **CDN** — Cloud Load Balancer 위에 cache 활성. 200+ edge POP.

---

## 2. 활성

```hcl
resource "google_compute_backend_bucket" "static" {
  name        = "static-cdn"
  bucket_name = google_storage_bucket.site.name
  enable_cdn  = true

  cdn_policy {
    cache_mode  = "CACHE_ALL_STATIC"
    default_ttl = 3600
    max_ttl     = 86400
    client_ttl  = 3600

    negative_caching = true
    negative_caching_policy { code = 404; ttl = 60 }
  }
}
```

→ static bucket = `enable_cdn = true`. 동적 backend = `cdn_policy`.

---

## 3. Cache modes

| Mode | 의미 |
| --- | --- |
| `USE_ORIGIN_HEADERS` | 옛 — origin 의 Cache-Control |
| `CACHE_ALL_STATIC` | 정적 파일 자동 cache |
| `FORCE_CACHE_ALL` | 무조건 cache (auth 주의) |

---

## 4. 비용

```
Cache fill (origin → edge): $0.01-0.04/GB
Cache egress (edge → user): region 별
  Asia:   $0.085 /GB
  US:     $0.085 /GB
HTTPS request: $0.0075 / 10K
```

CloudFront 와 비슷한 가격.

---

## 5. Invalidation

```bash
gcloud compute url-maps invalidate-cdn-cache myapp-url --path="/*"
```

→ deploy 후. 첫 1000/월 무료.

---

## 6. 함정

- origin Cache-Control 무시 (FORCE_CACHE_ALL)
- query string 의 cache key
- signed URL (private content)
- HTTP/3 (QUIC) 자동

---

## 7. 관련

- [[network]]
- [[cloud-lb]] — backend
- [[../storage/cloud-storage]] — backend bucket
