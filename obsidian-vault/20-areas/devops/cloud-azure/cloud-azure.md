---
title: "Azure — Cloud Hub"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:00:00+09:00
tags:
  - devops
  - cloud
  - azure
  - hub
---

# Azure — Cloud Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.1.0 | 2026-05-15 | engineering-agent/tech-lead | 카테고리별 서비스 분리 hub |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 최초 placeholder |

**[[../devops|↑ devops]]**

> Microsoft 의 클라우드. Windows / .NET / Active Directory / Office 통합 강점. 한국 기업 (제조 / 대기업) 사용 ↑.

---

## 1. 카테고리

| 카테고리 | 폴더 | 핵심 서비스 |
| --- | --- | --- |
| Compute | [[compute/compute]] | VM / AKS / Container Apps / Functions |
| Storage | [[storage/storage]] | Blob / Files |
| Database | [[database/database]] | Azure SQL / Cosmos DB / Cache for Redis |
| Network | [[network/network]] | VNet / App Gateway / Front Door / DNS |
| Security / IAM | [[security/security]] | Entra ID / Key Vault |
| Messaging | [[messaging/messaging]] | Service Bus / Event Grid |
| Observability | [[observability/observability]] | Monitor / Application Insights |

---

## 2. Azure 의 위계

```
Management Group   (회사 전체 정책)
  └── Subscription  (결제 + 격리)
        └── Resource Group  (응용 / 환경 단위)
              └── Resource    (VM / Storage / ...)
```

- AWS Account ≈ Subscription
- 모든 자원 = Resource Group 안 (필수)
- Tag — 비용 분석 / 정책

한국 region: `Korea Central` (Seoul), `Korea South` (Busan).

---

## 3. 시작

```bash
# Azure CLI
brew install azure-cli                              # macOS
sudo apt install azure-cli                          # Ubuntu (Microsoft repo)

az login
az account show
az account set --subscription "Pay-As-You-Go"
az group create -n myapp-prod -l koreacentral
```

---

## 4. AWS / GCP 와 매핑

| AWS | GCP | Azure |
| --- | --- | --- |
| EC2 | GCE | VM |
| ECS | Cloud Run | Container Apps |
| EKS | GKE | AKS |
| Lambda | Cloud Functions | Functions |
| S3 | Cloud Storage | Blob Storage |
| EBS | Persistent Disk | Managed Disks |
| RDS | Cloud SQL | Azure SQL / Postgres / MySQL |
| Aurora | AlloyDB | Hyperscale (옛) |
| DynamoDB | Firestore | Cosmos DB |
| Redshift | BigQuery | Synapse |
| VPC | VPC | VNet |
| ALB | Cloud LB | App Gateway / Load Balancer |
| CloudFront | Cloud CDN | Front Door / CDN |
| Route 53 | Cloud DNS | DNS |
| IAM | IAM | Entra ID (옛 Azure AD) + RBAC |
| KMS | Cloud KMS | Key Vault |
| Secrets Manager | Secret Manager | Key Vault Secrets |
| SQS / SNS | Pub/Sub | Service Bus / Event Grid / Event Hubs |
| CloudWatch | Operations Suite | Monitor + App Insights |
| CloudTrail | Audit Logs | Activity Log |

---

## 5. 강점

- **Windows / .NET** 통합
- **Active Directory** = Entra ID (회사 SSO 표준)
- **Office 365** 통합
- **Hybrid cloud** (Azure Arc) — on-premise 와의 연결 강함
- **Cosmos DB** — multi-model + multi-region

---

## 6. 비용 절약

- **Reserved Instance** — 1년/3년 약정 (40-72%)
- **Spot VM** — 90% 할인
- **Azure Hybrid Benefit** — Windows / SQL Server 라이선스 재사용
- **Dev/Test** subscription — 비-production 할인
- **Cost Management** + Budgets

---

## 7. IaC

| 도구 | 의미 |
| --- | --- |
| **ARM Template** | JSON (옛) |
| **Bicep** | ARM 의 친화적 DSL (Azure 권장) |
| **Terraform** | 표준 |
| **Pulumi** | 코드 |

```bash
# Bicep
az bicep build --file main.bicep
az deployment group create -g myapp -f main.bicep
```

---

## 8. 학습 자료

- **Azure 공식 docs** — learn.microsoft.com/azure
- **Microsoft Learn** — 무료 + 자격증
- **Azure Architecture Center** — Well-Architected
- **Azure CLI Reference**

---

## 9. 관련

- [[../cloud-aws/cloud-aws]] / [[../cloud-gcp/cloud-gcp]] / [[../cloud-oci/cloud-oci]]
- [[../kubernetes/kubernetes]] — AKS
- [[../devops|↑ devops]]
