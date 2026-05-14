---
title: "GCP Compute (Hub)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:05:00+09:00
tags:
  - gcp
  - compute
  - hub
---

# GCP Compute (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 카테고리 hub |

**[[../cloud-gcp|↑ GCP]]**

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **Compute Engine (GCE)** | [[gce]] | VM — AWS EC2 동등 |
| **GKE** | [[gke]] | Kubernetes — 사실상 최고 K8s |
| **Cloud Run** | [[cloud-run]] | Serverless container — 가장 권장 시작점 |

추가 (별도 노트 예정):
- **Cloud Functions** — 함수형 (Lambda 동등)
- **App Engine** — PaaS (옛)
- **Batch** — HPC

---

## 선택

| 시나리오 | 추천 |
| --- | --- |
| Container 단순 | **Cloud Run** |
| K8s 표준 | **GKE** |
| 전통 응용 / DB | **GCE** |
| 짧은 함수 | Cloud Functions |
| ML training | Vertex AI / GKE |

---

## 관련

- [[../cloud-gcp|↑ GCP]]
- [[../network/vpc]]
