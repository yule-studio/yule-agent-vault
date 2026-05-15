---
title: "k3s — lightweight kubernetes"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:36:00+09:00
tags: [area, devops, k3s, kubernetes]
---

# k3s — lightweight kubernetes

**[[../devops|↑ devops]]**

---

## 1. 왜 k3s

- 일반 k8s 너무 무거움 (etcd, kube-apiserver, scheduler, controller-manager 분리, 메모리 GB+).
- k3s = Rancher 가 만든 lightweight (single binary 60MB, 메모리 512MB).
- edge / IoT / small-scale / homelab / CI 에 적합.

→ **production k8s 보다는 작은 cluster / edge**.

---

## 2. 일반 k8s 와 차이

| | k8s | k3s |
| --- | --- | --- |
| 크기 | 1GB+ | 60MB binary |
| 메모리 | 1GB+ per node | 512MB |
| storage | etcd | SQLite (default) / etcd / external DB |
| ingress | manual install | Traefik 내장 |
| LB | manual / cloud | klipper-lb 내장 |
| 설치 | kubeadm / EKS 등 | `curl ... | sh` |
| 사용처 | production | edge / dev / homelab |

---

## 3. 설치

```bash
# server (master)
curl -sfL https://get.k3s.io | sh -

# worker (agent) — server token 필요
curl -sfL https://get.k3s.io | K3S_URL=https://server:6443 K3S_TOKEN=xxx sh -

# kubectl (server)
sudo k3s kubectl get nodes
# 또는 alias
echo 'alias kubectl="sudo k3s kubectl"' >> ~/.bashrc

# kubeconfig
sudo cat /etc/rancher/k3s/k3s.yaml > ~/.kube/config
sed -i 's/127.0.0.1/<server-ip>/' ~/.kube/config
```

---

## 4. 비활성 옵션

```bash
# server 설치 시 옵션
curl -sfL https://get.k3s.io | sh -s - server \
    --disable=traefik \                # 자체 ingress 사용 시
    --disable=servicelb \               # MetalLB 등 사용 시
    --disable=local-storage \           # 다른 storage 사용 시
    --write-kubeconfig-mode=644 \
    --tls-san=myserver.example.com
```

---

## 5. HA (High Availability)

```bash
# server 1: 첫 server (embedded etcd)
curl -sfL https://get.k3s.io | sh -s - server \
    --cluster-init \
    --token=mysecret

# server 2, 3: join
curl -sfL https://get.k3s.io | sh -s - server \
    --server=https://server1:6443 \
    --token=mysecret
```

→ 3 server (etcd quorum) + N agent 권장.

---

## 6. external DB (MySQL/Postgres)

```bash
curl -sfL https://get.k3s.io | sh -s - server \
    --datastore-endpoint="postgres://user:pass@host/db?sslmode=disable"
```

→ embedded etcd 대신 외부 DB. cluster 백업 / 복구 쉬움.

---

## 7. 사용

기본 k8s 와 동일 — `kubectl`, manifest, helm 등.

```bash
kubectl get nodes
kubectl get pods -A
helm install nginx bitnami/nginx
```

→ 99% 호환. **CRD, RBAC, NetworkPolicy 모두 동작**.

---

## 8. 자주 쓰는 case

| Case | 무엇 |
| --- | --- |
| **homelab** | RPi 4 + k3s — 학습 |
| **edge IoT** | 매장 / 공장 / 차량 |
| **dev cluster** | laptop 에 minikube 대신 |
| **GitOps demo** | ArgoCD + k3s |
| **CI** | k3s in CI for e2e |

→ **production 대규모는 EKS/GKE/AKS** 권장.

---

## 9. 함정

1. **SQLite default → single node만** — HA 면 etcd / external DB.
2. **Traefik 내장이 nginx-ingress 와 충돌** — disable=traefik.
3. **resource 작아도 일부 manifest 무거움** — Helm chart 가 prod 가정.
4. **upgrade in-place** — backup 필수.
5. **agent token rotation** — 정기 변경 필요.

---

## 10. 대안

| | 무엇 |
| --- | --- |
| **kind** | docker-in-docker k8s (CI 용) |
| **minikube** | VM 기반, full k8s |
| **microk8s** | Canonical, snap 설치, k3s 경쟁 |
| **k0s** | Mirantis, 비슷 |
| **kubeadm** | full k8s, manual |

→ **homelab / edge = k3s**, **CI = kind**, **dev = minikube**.

---

## 11. 관련

- [[../devops|↑ devops]]
- [[../kubernetes/kubernetes|↗ kubernetes]]
- [[../kubernetes/managed-vs-self-hosted|↗ managed vs self-hosted]]
