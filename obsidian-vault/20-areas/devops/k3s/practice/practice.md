---
title: "k3s — 실습"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T09:27:00+09:00
tags: [devops, k3s, practice]
---

# k3s — 실습

**[[../k3s|↑ k3s]]**

---

## 실습 시리즈

1. [[01-single-node-bootstrap]] — single node + first app
2. [[02-ha-3-server-cluster]] — embedded etcd HA + HAProxy LB
3. [[03-longhorn-storage]] — Longhorn 설치 + PVC + backup S3
4. [[04-argocd-gitops]] — ArgoCD App-of-Apps + ApplicationSet
5. [[05-edge-fleet-multistore]] — Fleet 으로 매장 cluster fleet 관리

---

## 사전 준비

| Lab | 환경 |
| --- | --- |
| 01 | Ubuntu VM 1대 (4GB RAM) |
| 02 | Ubuntu VM 3대 + LB VM 1대 |
| 03 | Lab 02 위에서 + S3 bucket |
| 04 | Lab 02 위에서 + GitHub repo |
| 05 | 3 가상 cluster (k3d) + Rancher VM |

---

## 학습 순서

```
Day 1: 01 (single node + first app + first PVC)
Day 2: 02 (HA + HAProxy)
Day 3: 03 (Longhorn replica 3 + backup)
Day 4: 04 (ArgoCD root app + 5 sub-app)
Day 5: 05 (3 cluster + Fleet GitRepo + targeted deploy)
```

---

## 검증 기준

각 lab 의 마지막에:
- ☐ 모든 step success
- ☐ failure 시 복구 절차 실행
- ☐ runbook 업데이트
- ☐ commit (이 vault 의 lab 결과)

---

## 참고 자료

- k3s docs: https://docs.k3s.io/
- ArgoCD docs: https://argo-cd.readthedocs.io/
- Longhorn: https://longhorn.io/docs/
- Fleet: https://fleet.rancher.io/
