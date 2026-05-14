---
title: "GCP Cloud DNS"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:20:00+09:00
tags:
  - gcp
  - network
  - dns
---

# GCP Cloud DNS

**[[network|↑ Network]]** · **[[../cloud-gcp|↑↑ GCP]]**

---

## 1. 한 줄

매니지드 DNS — AWS Route 53 동등. 100% SLA + 글로벌 anycast.

---

## 2. 사용

```bash
gcloud dns managed-zones create example \
  --dns-name=example.com. \
  --description="example.com"

gcloud dns record-sets transaction start --zone=example
gcloud dns record-sets transaction add 1.2.3.4 \
  --name=api.example.com. --ttl=300 --type=A --zone=example
gcloud dns record-sets transaction execute --zone=example
```

```hcl
resource "google_dns_managed_zone" "main" {
  name        = "example"
  dns_name    = "example.com."
  description = "main zone"
  dnssec_config {
    state = "on"
  }
}

resource "google_dns_record_set" "api" {
  name         = "api.example.com."
  managed_zone = google_dns_managed_zone.main.name
  type         = "A"
  ttl          = 300
  rrdatas      = [google_compute_global_address.lb.address]
}
```

---

## 3. Routing Policies

- **WRR (Weighted Round Robin)** — canary
- **Geo Location** — 국가별
- **Failover** — primary / backup

```hcl
resource "google_dns_record_set" "api_geo" {
  routing_policy {
    geo {
      location = "asia-northeast3"
      rrdatas  = ["1.2.3.4"]
    }
    geo {
      location = "us-central1"
      rrdatas  = ["5.6.7.8"]
    }
  }
}
```

---

## 4. Private Zone

```hcl
resource "google_dns_managed_zone" "internal" {
  name     = "internal"
  dns_name = "internal.example.com."
  visibility = "private"
  private_visibility_config {
    networks {
      network_url = google_compute_network.vpc.id
    }
  }
}
```

→ VPC 안에서만 resolve.

---

## 5. 비용

```
Zone: $0.20 / month / zone (첫 25)
Queries: $0.40 / 1M (첫 1B/월), 이후 $0.20
DNSSEC: + KMS 비용
```

---

## 6. 함정

- TTL 짧으면 query 비용 ↑
- DNSSEC 잘못 = lockout
- registrar 의 NS 변경 = 전파 시간

---

## 7. 관련

- [[network]]
- [[cloud-lb]] — target
- [[../../../computer-science/network/dns/dns|↗ DNS 이론]]
