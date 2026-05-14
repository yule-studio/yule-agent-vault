---
title: "GCP Compute Engine (GCE)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:10:00+09:00
tags:
  - gcp
  - compute
  - gce
  - vm
---

# GCP Compute Engine (GCE)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | GCE 개념 + 사용 |

**[[compute|↑ Compute]]** · **[[../cloud-gcp|↑↑ GCP]]**

---

## 1. 한 줄

GCP 의 가상 머신 — AWS EC2 동등. **live migration** + **sustained use discount** 가 강점.

---

## 2. Machine Type

| Family | 용도 |
| --- | --- |
| **e2** | 범용 (low cost) |
| **n2 / n2d** | 범용 (Intel / AMD) |
| **n4** | 최신 |
| **c3 / c4** | CPU intensive |
| **m3** | RAM 많음 |
| **a2 / g2** | GPU |
| **t2d** | ARM (Tau T2D) |

기본 = e2-medium / e2-standard. 운영 = n2-standard.

---

## 3. 설치

### 3.1 gcloud

```bash
gcloud compute instances create myapp \
  --machine-type=e2-medium \
  --zone=asia-northeast3-a \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --boot-disk-size=20GB \
  --boot-disk-type=pd-balanced

gcloud compute ssh myapp --zone=asia-northeast3-a
gcloud compute instances stop / start / delete myapp
```

### 3.2 Terraform

```hcl
resource "google_compute_instance" "myapp" {
  name         = "myapp"
  machine_type = "e2-medium"
  zone         = "asia-northeast3-a"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-12"
      size  = 20
      type  = "pd-balanced"
    }
  }

  network_interface {
    network    = google_compute_network.vpc.id
    subnetwork = google_compute_subnetwork.private.id
    # public IP 없음 (private)
  }

  service_account {
    email  = google_service_account.app.email
    scopes = ["cloud-platform"]
  }

  metadata_startup_script = file("startup.sh")
}
```

---

## 4. 접속 (OS Login 권장)

```bash
# OS Login 활성 (key SSH 없이 IAM)
gcloud compute project-info add-metadata --metadata enable-oslogin=TRUE

# 접속
gcloud compute ssh myapp                     # 자동 SSH key 관리

# IAP tunnel (방화벽 IP 화이트리스트 불필요)
gcloud compute ssh myapp --tunnel-through-iap
```

→ IAM 으로 누가 SSH 접속 가능한지 관리. AWS SSM 비슷.

---

## 5. Disk

```hcl
resource "google_compute_disk" "data" {
  name  = "data"
  type  = "pd-ssd"
  zone  = "asia-northeast3-a"
  size  = 100
}

resource "google_compute_attached_disk" "data" {
  disk     = google_compute_disk.data.id
  instance = google_compute_instance.myapp.id
}
```

| Type | IOPS | 용도 |
| --- | --- | --- |
| pd-standard | HDD | cold / 대용량 |
| pd-balanced | SSD 균형 | 기본 |
| pd-ssd | SSD 고성능 | DB |
| pd-extreme | 최고 | 거대 DB |
| hyperdisk | 새 — IOPS/throughput 독립 조정 | 신규 권장 |

---

## 6. Instance Group / Managed Instance Group (MIG)

```hcl
resource "google_compute_instance_template" "app" { ... }

resource "google_compute_region_instance_group_manager" "app" {
  name                = "app-mig"
  region              = "asia-northeast3"
  base_instance_name  = "app"
  target_size         = 3

  version {
    instance_template = google_compute_instance_template.app.id
  }

  auto_healing_policies {
    health_check      = google_compute_health_check.app.id
    initial_delay_sec = 300
  }
}

resource "google_compute_region_autoscaler" "app" {
  region = "asia-northeast3"
  target = google_compute_region_instance_group_manager.app.id

  autoscaling_policy {
    min_replicas    = 2
    max_replicas    = 10
    cpu_utilization {
      target = 0.6
    }
  }
}
```

→ AWS Auto Scaling Group 동등. region-wide 가능 (multi-zone 자동).

---

## 7. Live Migration

GCP 의 특별한 기능 — 유지보수 시 VM 을 **다른 host 로 라이브 마이그**. 응용은 모름.
- AWS EC2 = 보통 stop / restart 필요
- GCE = 무중단

---

## 8. 가격 모델

| 모델 | 의미 |
| --- | --- |
| **On-Demand** | 사용한 만큼 |
| **Sustained Use Discount (SUD)** | 한 달 25%+ 사용 시 자동 할인 (~30%) |
| **Committed Use Discount (CUD)** | 1년 / 3년 약정 (40-60%) |
| **Spot VM** | 60-90% 할인 + 강제 종료 |
| **Preemptible (옛)** | Spot 의 옛 이름 |

→ SUD 가 자동으로 적용되어 EC2 보다 단순한 가격 구조.

---

## 9. 비용 (Seoul)

| 인스턴스 | $/시간 | $/월 |
| --- | --- | --- |
| e2-micro (free tier) | ~$0.008 | ~$6 |
| e2-medium | ~$0.034 | ~$25 |
| n2-standard-2 | ~$0.107 | ~$78 |
| n2-standard-4 | ~$0.213 | ~$155 |

SUD 자동 적용 시 ~ 30% 할인.

---

## 10. 사용 시나리오

- 전통 응용 / 큰 서버
- DB self-managed
- ML training (GPU)
- CI / 빌드
- VPN bastion / Cloud NAT

대부분 = container 표준 → **GKE / Cloud Run** 우선.

---

## 11. 함정

### 11.1 zonal vs regional
GCE = zonal. MIG 는 regional 가능 (zone 자동 spread).

### 11.2 default network
모든 VM 이 default VPC + 인터넷 노출. private 별도 VPC.

### 11.3 OS Login + IAM
키 관리 + IAM. 옛 SSH key metadata 보다 안전.

### 11.4 preemptible / spot
24h 후 자동 종료 (preemptible). Spot 은 단순 60-90% 할인 + 임의 종료.

### 11.5 SUD 의 의외 비용
부분 사용 시 SUD 적용 X. 한 달 80%+ = 거의 30% 자동 할인.

---

## 12. 학습 자료

- GCP Compute Engine docs
- **Coursera — Architecting with Google Compute Engine**

---

## 13. 관련

- [[compute]] — Compute hub
- [[gke]] / [[cloud-run]] — container 대안
- [[../storage/persistent-disk]]
- [[../network/vpc]]
