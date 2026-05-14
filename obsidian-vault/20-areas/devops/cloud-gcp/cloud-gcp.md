---
title: "GCP — Cloud Hub"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:00:00+09:00
tags:
  - devops
  - cloud
  - gcp
  - hub
---

# GCP — Cloud Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.1.0 | 2026-05-14 | engineering-agent/tech-lead | 카테고리별 서비스 분리 hub |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 최초 placeholder |

**[[../devops|↑ devops]]**

> Google Cloud Platform — 데이터 / AI 가 강점. Kubernetes / BigQuery / Vertex AI 의 본가.

---

## 1. 카테고리

| 카테고리 | 폴더 | 핵심 서비스 |
| --- | --- | --- |
| Compute | [[compute/compute]] | GCE / GKE / Cloud Run |
| Storage | [[storage/storage]] | Cloud Storage / Persistent Disk |
| Database | [[database/database]] | Cloud SQL / Spanner / Firestore / BigQuery |
| Network | [[network/network]] | VPC / Cloud LB / Cloud CDN / Cloud DNS |
| Security / IAM | [[security/security]] | IAM / Secret Manager / Cloud KMS |
| Messaging | [[messaging/messaging]] | Pub/Sub |
| Observability | [[observability/observability]] | Cloud Logging / Monitoring |

---

## 2. 시작

```bash
# gcloud CLI
curl https://sdk.cloud.google.com | bash
gcloud init                                # 인증 + project / region
gcloud auth login
gcloud auth application-default login      # SDK 용

# project 별
gcloud config set project my-project
gcloud config set compute/region asia-northeast3      # Seoul
```

---

## 3. 핵심 구조

| 개념 | 의미 |
| --- | --- |
| **Organization** | 최상위 (Google Workspace + GCP) |
| **Folder** | 부서 / 환경 |
| **Project** | 자원 / 결제 단위 — 모든 자원 = 1 project 소속 |
| **Region / Zone** | 지역 / AZ |

→ AWS account ≈ GCP project. 환경 별 분리 표준.

한국 region: `asia-northeast3` (Seoul) — Compute / GKE / Cloud SQL 등 대부분.

---

## 4. AWS vs GCP 매핑

| AWS | GCP |
| --- | --- |
| EC2 | Compute Engine (GCE) |
| ECS / EKS | GKE / Cloud Run |
| Lambda | Cloud Functions / Cloud Run |
| S3 | Cloud Storage |
| EBS | Persistent Disk |
| RDS | Cloud SQL |
| Aurora | AlloyDB / Spanner |
| DynamoDB | Firestore / Bigtable |
| Redshift | BigQuery |
| VPC | VPC (multi-region 가능) |
| ALB | Cloud Load Balancing (Global) |
| CloudFront | Cloud CDN |
| Route 53 | Cloud DNS |
| IAM | IAM + Workload Identity |
| KMS | Cloud KMS |
| Secrets Manager | Secret Manager |
| SQS / SNS | Pub/Sub |
| CloudWatch | Cloud Logging + Monitoring |
| CloudTrail | Cloud Audit Logs |

---

## 5. 비용 절약

- **Sustained Use Discount** — 한 달 거의 사용 시 자동 30% 할인 (GCE)
- **Committed Use Discount (CUD)** — 1년/3년 약정
- **Preemptible / Spot VM** — 24h 한계 / 강제 종료
- **Free Tier** — f1-micro VM, 5GB Cloud Storage 등
- **BigQuery flat-rate** — 약정 slot

---

## 6. IaC

| 도구 | 의미 |
| --- | --- |
| **Deployment Manager** | GCP 공식 (옛, deprecated) |
| **Terraform** | 표준 |
| **Pulumi** | 코드로 |
| **Config Connector** | K8s CRD 로 GCP 자원 |

```bash
terraform init && terraform apply
```

---

## 7. 강점

- **Kubernetes 본가** — GKE 가 최고 K8s 경험
- **BigQuery** — 분석 / DW 의 표준
- **Vertex AI / Gemini** — ML 통합
- **글로벌 네트워크** — 자체 backbone (Premium tier)
- **Spanner** — 글로벌 일관성 RDB

---

## 8. 학습 자료

- **GCP 공식 docs** — cloud.google.com/docs
- **Google Cloud Architecture Framework** — Well-Architected
- **Coursera GCP** — 공식 과정 + 자격증
- **Qwiklabs** — 실습

---

## 9. 관련

- [[../cloud-aws/cloud-aws]] / [[../cloud-azure/cloud-azure]] / [[../cloud-oci/cloud-oci]]
- [[../kubernetes/kubernetes]] — GKE
- [[../devops|↑ devops]]
