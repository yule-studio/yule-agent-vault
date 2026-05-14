---
title: "Azure Compute (Hub)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:05:00+09:00
tags:
  - azure
  - compute
  - hub
---

# Azure Compute (Hub)

**[[../cloud-azure|↑ Azure]]**

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **Virtual Machine (VM)** | [[vm]] | IaaS (AWS EC2 동등) |
| **AKS** | [[aks]] | Kubernetes |
| **Container Apps** | [[container-apps]] | Serverless container (Cloud Run 동등) |
| **Functions** | [[functions]] | Serverless function (Lambda 동등) |

추가:
- **App Service** — managed PaaS (전통)
- **Batch** — HPC
- **Service Fabric** — 옛 microservice

---

## 선택

| 시나리오 | 추천 |
| --- | --- |
| 단순 container | Container Apps |
| K8s 표준 | AKS |
| 전통 .NET 응용 | App Service |
| 짧은 함수 | Functions |
| 큰 RAM / 특수 OS | VM |

---

## 관련
- [[../cloud-azure|↑ Azure]]
- [[../network/vnet]]
