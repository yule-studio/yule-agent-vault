---
title: "Managed vs Self-hosted Kubernetes"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:00:00+09:00
tags: [devops, kubernetes, managed]
---

# Managed vs Self-hosted Kubernetes

**[[kubernetes|↑ k8s]]**

---

## 1. 비교

| | **Managed** ★ | Self-hosted (kubeadm) |
| --- | --- | --- |
| control plane | cloud 관리 | 직접 운영 |
| node | cloud node (관리) 또는 self | self |
| 업그레이드 | 1 클릭 | 수동 / 위험 |
| 비용 | $0.10/h control + node | node 만 |
| 학습 | 쉬움 | 매우 어려움 |
| 본 vault | ★ (시작 시) | 특별 요구 시 |

---

## 2. cloud managed 비교

| | **EKS** | **GKE** ★ | **AKS** | **OKE** |
| --- | --- | --- | --- | --- |
| 출시 | 2018 | 2014 (k8s 발명사) | 2018 | 2018 |
| 자동 업그레이드 | Manual + auto | Auto + release channels | Auto | Auto |
| 가격 | $0.10/h + nodes | $0.10/h (Standard) / 무료 (Autopilot) | $0 control plane | $0 control plane |
| Autopilot | X | O (서버 추상) ★ | X | X |
| 추천 | AWS 환경 | 시작 / 빠른 신기능 | Azure 환경 | OCI 환경 |

→ 본 vault: **GKE** (가장 성숙) — 단, 회사가 AWS 면 EKS.

---

## 3. Self-hosted 시기

- 매우 큰 cluster (1000+ 노드).
- on-premise 데이터 센터.
- 특수 customization (CNI / runtime).
- compliance (FIPS / air-gap).

→ 시작 시는 항상 managed.

---

## 4. 관련

- [[kubernetes|↑ k8s]]
- [[../cloud-aws/cloud-aws|↗ AWS]]
- [[../cloud-gcp/cloud-gcp|↗ GCP]]
- [[../cloud-azure/cloud-azure|↗ Azure]]
- [[../k3s/k3s|↗ k3s (edge)]]
