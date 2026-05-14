---
title: "GCP Cloud Run — Serverless Container"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:20:00+09:00
tags:
  - gcp
  - compute
  - cloud-run
  - serverless
---

# GCP Cloud Run

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Cloud Run 개념 + 사용 |

**[[compute|↑ Compute]]** · **[[../cloud-gcp|↑↑ GCP]]**

---

## 1. 한 줄

**serverless container** — Docker image 만 주면 HTTP / event-driven 으로 자동 실행 + scale-to-zero.

GCP 의 가장 권장 시작점 — K8s 없이 container 운영.

---

## 2. 왜

- 응용을 container 화 + 1 명령 배포 → 자동 HTTPS endpoint
- 사용 안 하면 0 cost (scale-to-zero)
- 자동 scale 1 → 수천
- K8s 학습 X
- Knative 기반 (표준)

대안:
- **Cloud Functions** — code 자체 (image X)
- **GKE Autopilot** — K8s feature 필요
- **App Engine** — PaaS (옛)

---

## 3. 두 모드

| | Service | Job |
| --- | --- | --- |
| 의도 | HTTP 응답 | batch 작업 |
| trigger | request | manual / scheduled |
| 시간 한계 | 60분 | 24시간 |

→ 일반 API = Service. 일회성 / batch = Job.

---

## 4. 설치

### 4.1 1 명령 배포

```bash
# 소스에서 직접 (Cloud Build 가 빌드)
gcloud run deploy myapp \
  --source=. \
  --region=asia-northeast3 \
  --allow-unauthenticated

# 또는 이미 있는 image
gcloud run deploy myapp \
  --image=gcr.io/PROJECT/myapp:v1 \
  --region=asia-northeast3 \
  --memory=512Mi \
  --cpu=1 \
  --min-instances=0 \
  --max-instances=100 \
  --concurrency=80 \
  --timeout=300
```

→ 즉시 HTTPS endpoint (`https://myapp-xxx.asia-northeast3.run.app`).

### 4.2 Terraform

```hcl
resource "google_cloud_run_v2_service" "myapp" {
  name     = "myapp"
  location = "asia-northeast3"

  template {
    scaling {
      min_instance_count = 0
      max_instance_count = 100
    }
    containers {
      image = "asia-northeast3-docker.pkg.dev/PROJECT/repo/myapp:v1"
      resources {
        limits = { cpu = "1", memory = "512Mi" }
      }
      env {
        name  = "DB_HOST"
        value = "cloud-sql-instance"
      }
    }
    service_account = google_service_account.app.email
    timeout         = "300s"
  }
}

resource "google_cloud_run_v2_service_iam_binding" "public" {
  location = google_cloud_run_v2_service.myapp.location
  name     = google_cloud_run_v2_service.myapp.name
  role     = "roles/run.invoker"
  members  = ["allUsers"]
}
```

---

## 5. 핵심 설정

| 설정 | 의미 |
| --- | --- |
| **CPU** | 0.5 / 1 / 2 / 4 / 8 |
| **Memory** | 128 Mi - 32 Gi |
| **Concurrency** | 한 인스턴스가 처리하는 동시 요청 (default 80) |
| **min instance** | 0 = scale-to-zero, > 0 = warm |
| **max instance** | 상한 |
| **timeout** | 요청 max 60분 |
| **CPU always allocated / on-request** | startup / 정상 시 CPU 항상 vs 요청 시 |

---

## 6. Cold Start

```
request 도착 → instance 0 → 새 인스턴스 boot → 응용 시작 → 요청 처리
시간: 100ms (작은) ~ 수 초 (큰 JVM)
```

회피:
- min instances = 1 (작은 비용)
- 작은 image
- CPU always allocated (init 빠름)
- startup CPU boost (자동)

---

## 7. Trigger

| | |
| --- | --- |
| HTTPS | 자동 endpoint |
| Pub/Sub push | topic 메시지 → request |
| Cloud Storage event | object 생성 / 삭제 |
| EventArc | 더 다양한 GCP event |
| Cloud Scheduler | cron-like |

→ event-driven 자연스럽게.

---

## 8. 가격 (Seoul)

```
vCPU:    $0.024 / vCPU·시간 (request 처리 중만)
Memory:  $0.0025 / GB·시간
Request: $0.40 / 1M
Networking: outbound 별도

Free tier (월):
  2M request / 360K vCPU·second / 180K GB·second
```

→ 작은 사이트 = 거의 무료. 큰 사용량 = ECS/GKE 와 비교 후 결정.

---

## 9. Cloud Run + 데이터베이스

```hcl
# Cloud SQL 접근
resource "google_cloud_run_v2_service" "myapp" {
  template {
    containers { ... }
    vpc_access {
      connector = google_vpc_access_connector.myapp.id
      egress    = "PRIVATE_RANGES_ONLY"
    }
  }
}
```

또는 Cloud SQL **Direct VPC** (새, 더 빠름).

자세히 → [[../database/cloud-sql]]

---

## 10. 사용 시나리오

- API / 백엔드 (대부분)
- webhook
- 이벤트 처리 (Pub/Sub → Cloud Run)
- 짧은 배치 (Job)
- 정적 사이트 + 일부 동적

부적합:
- WebSocket persistent (옛 — 새 streaming OK)
- 매우 long-running (60분+)
- GPU (별도)
- raw TCP / UDP

---

## 11. 함정

### 11.1 max concurrency
default 80 — CPU intensive 응용엔 작게 (1).

### 11.2 cold start latency
critical path = min instance > 0.

### 11.3 Cloud SQL 의 connection
proxy + connection pool 신경. limit ↑.

### 11.4 stateless
in-memory state X. Redis / Firestore.

### 11.5 outbound egress 비용
인터넷 / cross-region.

### 11.6 timeout 60분 의 의미
HTTP 응답 시간. 더 길면 background task → Cloud Tasks.

---

## 12. 학습 자료

- GCP Cloud Run docs
- **Knative** (Cloud Run 의 토대)

---

## 13. 관련

- [[compute]] — Compute hub
- [[gce]] / [[gke]]
- [[../database/cloud-sql]]
- [[../messaging/pubsub]]
