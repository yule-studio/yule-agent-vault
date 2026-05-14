---
title: "OCI — Oracle Cloud Infrastructure (Hub)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T04:00:00+09:00
tags:
  - devops
  - cloud
  - oci
  - oracle-cloud
  - hub
---

# OCI — Oracle Cloud Infrastructure

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.1.0 | 2026-05-14 | engineering-agent/tech-lead | 카테고리별 서비스 분리 hub |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 최초 placeholder |

**[[../devops|↑ devops]]**

> Oracle 의 클라우드. **Autonomous Database** + **Always Free** + 가성비. ARM (Ampere) 무료 인스턴스 큼.

---

## 1. 카테고리

| 카테고리 | 폴더 | 핵심 서비스 |
| --- | --- | --- |
| Compute | [[compute/compute]] | Instance / OKE / Functions |
| Storage | [[storage/storage]] | Object Storage / Block Volume |
| Database | [[database/database]] | Autonomous DB / MySQL HeatWave |
| Network | [[network/network]] | VCN / LB / DNS |
| Security / IAM | [[security/security]] | IAM / Vault |
| Messaging | [[messaging/messaging]] | Streaming / Notifications |
| Observability | [[observability/observability]] | Monitoring / Logging |

---

## 2. OCI 의 위계

```
Tenancy            (= AWS account / Azure tenant)
  └── Compartment   (논리 격리 — 트리 가능)
        └── Resource
```

- **Compartment** = OCI 특유 — IAM policy 의 단위 (프로젝트 / 환경 / 팀)
- region: `ap-seoul-1` (Seoul), `ap-chuncheon-1` (Chuncheon)
- **Availability Domain (AD)** — AZ
- **Fault Domain (FD)** — AD 안 hardware 그룹

---

## 3. 시작

```bash
# OCI CLI
brew install oci-cli                  # macOS
oci setup config                      # 대화형 설정

# 또는 cloud shell (브라우저 안)
```

---

## 4. AWS / GCP / Azure 와 매핑

| AWS | GCP | Azure | OCI |
| --- | --- | --- | --- |
| EC2 | GCE | VM | **Compute Instance** |
| EKS | GKE | AKS | **OKE** (Oracle K8s) |
| Lambda | Cloud Functions | Functions | **Functions** (Fn) |
| S3 | Cloud Storage | Blob | **Object Storage** |
| EBS | Persistent Disk | Managed Disk | **Block Volume** |
| RDS | Cloud SQL | Azure SQL | **Autonomous DB** |
| Aurora | AlloyDB | Hyperscale | **Autonomous (Serverless)** |
| Redshift | BigQuery | Synapse | **Autonomous DW** |
| DynamoDB | Firestore | Cosmos DB | **NoSQL Database** |
| VPC | VPC | VNet | **VCN** |
| ELB | Cloud LB | LB | **Load Balancer** / **NLB** |
| CloudFront | Cloud CDN | Front Door | (3rd party) |
| Route 53 | Cloud DNS | DNS | **DNS** |
| IAM | IAM | RBAC / Entra ID | **IAM** + Compartment |
| KMS | Cloud KMS | Key Vault | **Vault** |
| SQS | Pub/Sub | Service Bus | **Streaming** + **Queue** |
| Kinesis | Pub/Sub | Event Hubs | **Streaming** |
| SNS | Pub/Sub | Event Grid | **Notifications** |
| CloudWatch | Operations | Monitor | **Monitoring** + **Logging** |

---

## 5. 강점

- **Always Free** — 평생 무료 tier 가 큼 (ARM 4 OCPU / 24 GB / 200 GB block 등)
- **Autonomous Database** — Oracle DB 의 자동 튜닝 / 패치
- **가격 경쟁력** — outbound bandwidth 10 TB 무료
- **Bare Metal** — full 베어메탈 인스턴스 (가상화 없음)
- **ARM Ampere** — 가성비 압도

---

## 6. Always Free

- AMD VM: 2 micro instance (1/8 OCPU, 1 GB)
- **ARM Ampere A1**: 4 OCPU / 24 GB RAM (조각 가능)
- Object Storage: 20 GB
- Block Volume: 200 GB
- Autonomous DB: 2 (20 GB 각)
- Load Balancer: 1 (10 Mbps)
- 외부 bandwidth: 10 TB/월

→ 사이드 프로젝트 / 학습 매력적.

---

## 7. IaC

```bash
# Terraform OCI provider
terraform init
terraform plan
terraform apply

# OCI Resource Manager — managed Terraform
```

```hcl
provider "oci" {
  tenancy_ocid     = "..."
  user_ocid        = "..."
  fingerprint      = "..."
  private_key_path = "..."
  region           = "ap-seoul-1"
}
```

---

## 8. 학습 자료

- **OCI Docs** — docs.oracle.com/iaas
- **Oracle University** — 무료 학습 + 자격증
- **OCI 한국 community**
- **terraform-provider-oci**

---

## 9. 관련

- [[../cloud-aws/cloud-aws]] / [[../cloud-gcp/cloud-gcp]] / [[../cloud-azure/cloud-azure]]
- [[../kubernetes/kubernetes]] — OKE
- [[../../../database/oracle/oracle|↗ Oracle DB]]
- [[../devops|↑ devops]]
