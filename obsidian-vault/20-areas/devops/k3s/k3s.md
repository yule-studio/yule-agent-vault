---
title: "k3s — lightweight kubernetes ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:36:00+09:00
tags: [area, devops, k3s, kubernetes]
---

# k3s — lightweight kubernetes ★

**[[../devops|↑ devops]]**

---

## 1. 왜 k3s

- 일반 k8s 너무 무거움 (etcd / apiserver / scheduler / controller-manager 분리, 메모리 GB+).
- k3s = Rancher (현 SUSE) lightweight (single binary < 70MB, RAM 512MB ~ 1GB).
- edge / IoT / small-scale / homelab / CI 에 적합.

→ **production k8s 보다는 작은 cluster / edge** 가 main use case.

---

## 2. 일반 k8s 와 차이

| | k8s | k3s |
| --- | --- | --- |
| 크기 | 1GB+ | < 70MB binary |
| 메모리 | 1GB+ per node | 512MB |
| storage | etcd | SQLite (default) / etcd / external DB |
| ingress | manual install | Traefik 내장 |
| LB | manual / cloud | klipper-lb 내장 |
| CNI | manual | flannel 내장 (VXLAN) |
| 설치 | kubeadm / EKS 등 | `curl ... | sh` |
| upgrade | 다단계 | `curl ... | sh` 다시 |
| 사용처 | production 대규모 | edge / dev / homelab / 중소 production |

---

## 3. 하위 영역

- [[concepts]] — architecture / single-binary 구조 / SQLite / kine
- [[installation]] — server / agent / token / config / systemd
- [[ha-mode]] — embedded etcd / external DB (Postgres/MySQL)
- [[networking]] — flannel / kube-proxy / Traefik / klipper-lb
- [[storage]] — local-path / Longhorn / external CSI
- [[upgrade-strategy]] — channel / 자동 upgrade controller
- [[backup-restore]] — etcd snapshot / SQLite backup / external DB
- [[gitops]] — k3s + ArgoCD / Fleet (Rancher)
- [[production-checklist]] — 운영 전 점검 (TLS / token / monitoring / RBAC)
- [[edge-usecases]] — RPi cluster / K3OS / 매장-공장 / vehicle
- [[k3d-multi-cluster]] — k3d (k3s-in-docker) 다중 cluster dev
- [[migration]] — k3s ↔ k8s 마이그레이션
- [[pitfalls]]
- [[practice/practice|practice]] — 5 hands-on lab

---

## 4. 학습 순서

1. Day 1: [[concepts]] + [[installation]] — single node k3s 띄우기
2. Day 2: [[networking]] + [[storage]] — Traefik / local-path 이해
3. Day 3: [[ha-mode]] — 3 server HA cluster
4. Day 4: [[gitops]] + [[upgrade-strategy]] — ArgoCD / 자동 upgrade
5. Day 5: [[production-checklist]] + [[backup-restore]] + [[edge-usecases]]

---

## 5. 자주 쓰는 case

| Case | 무엇 |
| --- | --- |
| **homelab** | RPi 4/5 cluster — 학습 / personal service |
| **edge IoT** | 매장 / 공장 / 차량 (single node + 자동복구) |
| **dev cluster** | laptop / VM 에 minikube 대신 |
| **GitOps demo** | ArgoCD + k3s |
| **CI** | k3s in CI 또는 [[k3d-multi-cluster|k3d]] for e2e |
| **중소 production** | 1-10 node, 비용 절감 |
| **air-gapped** | offline 환경 (선박, 군) |

→ **대규모 production 은 EKS/GKE/AKS** 권장.

---

## 6. k3s ecosystem

| | 무엇 |
| --- | --- |
| **k3s** | Lightweight k8s distribution |
| **k3d** | k3s in Docker (다중 cluster dev) |
| **k3sup** | k3s 설치 자동화 (Go binary) |
| **k3os** | k8s 만 위한 immutable OS |
| **Rancher** | k3s 의 enterprise management |
| **Fleet** | Rancher 의 GitOps |
| **autok3s** | cloud (AWS/Aliyun) 에 k3s 자동 배포 |
| **Harvester** | HCI (k3s + KubeVirt) |

---

## 7. 대안

| | 무엇 |
| --- | --- |
| **kind** | docker-in-docker k8s (CI 용, 빠름) |
| **minikube** | VM 기반, full k8s |
| **microk8s** | Canonical, snap 설치, k3s 경쟁 |
| **k0s** | Mirantis, 비슷 |
| **kubeadm** | full k8s, manual |
| **Talos Linux** | API-driven OS for k8s |

→ **homelab / edge / 중소 prod = k3s**, **CI = kind**, **dev = minikube / k3d**.

---

## 8. 관련

- [[../devops|↑ devops]]
- [[../kubernetes/kubernetes|↗ kubernetes]]
- [[../kubernetes/managed-vs-self-hosted|↗ managed vs self-hosted]]
- [[../cicd/argocd-cd|↗ ArgoCD]]
