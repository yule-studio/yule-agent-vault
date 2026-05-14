---
title: "GCP VPC"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:05:00+09:00
tags:
  - gcp
  - network
  - vpc
---

# GCP VPC

**[[network|↑ Network]]** · **[[../cloud-gcp|↑↑ GCP]]**

---

## 1. 핵심 차이 (AWS 와)

| | AWS VPC | GCP VPC |
| --- | --- | --- |
| 범위 | region | **global** |
| Subnet | AZ 단위 | region 단위 (multi-zone) |
| Inter-region | peering / TGW | 같은 VPC 안 |
| Default network | 있음 (지움 권장) | 있음 (지움 권장) |

→ GCP VPC 가 더 단순한 multi-region.

---

## 2. 생성

```bash
gcloud compute networks create myapp-vpc --subnet-mode=custom

gcloud compute networks subnets create gke \
  --network=myapp-vpc \
  --region=asia-northeast3 \
  --range=10.0.0.0/20 \
  --secondary-range pods=10.4.0.0/14,services=10.0.16.0/20
```

```hcl
resource "google_compute_network" "vpc" {
  name                    = "myapp-vpc"
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "private" {
  name          = "private"
  region        = "asia-northeast3"
  network       = google_compute_network.vpc.id
  ip_cidr_range = "10.0.0.0/20"
  private_ip_google_access = true

  secondary_ip_range {
    range_name    = "pods"
    ip_cidr_range = "10.4.0.0/14"
  }
  secondary_ip_range {
    range_name    = "services"
    ip_cidr_range = "10.0.16.0/20"
  }
}
```

---

## 3. Firewall

GCP firewall = **VPC 단위** (AWS SG = 인스턴스 단위 다름).

```hcl
resource "google_compute_firewall" "allow_ssh_iap" {
  name    = "allow-ssh-iap"
  network = google_compute_network.vpc.name

  allow {
    protocol = "tcp"
    ports    = ["22"]
  }
  source_ranges = ["35.235.240.0/20"]      # IAP CIDR
  target_tags   = ["ssh-allowed"]
}
```

태그 / SA / IP 로 매칭.

---

## 4. Cloud NAT

```hcl
resource "google_compute_router" "router" {
  name    = "myapp-router"
  region  = "asia-northeast3"
  network = google_compute_network.vpc.id
}

resource "google_compute_router_nat" "nat" {
  name                               = "myapp-nat"
  router                             = google_compute_router.router.name
  region                             = "asia-northeast3"
  nat_ip_allocate_option             = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"
}
```

→ private → 인터넷 outbound. AWS NAT Gateway 동등.

---

## 5. VPC Peering / Network Connectivity

- **VPC Peering** — 두 VPC 1:1
- **Network Connectivity Center** — hub-and-spoke
- **VPN / Interconnect** — on-prem

---

## 6. Private Service Connect (PSC)

PrivateLink 동등 — Cloud SQL / 다른 VPC 의 서비스를 private 으로.

---

## 7. 함정

- default network 삭제 권장 (보안)
- firewall rule = VPC 전체. 잘못 = 큰 영향
- secondary range — GKE 용 미리 계획 (큰 /14)
- region 간 traffic 가격 (egress)

---

## 8. 관련

- [[network]]
- [[../compute/gce]] / [[../compute/gke]]
