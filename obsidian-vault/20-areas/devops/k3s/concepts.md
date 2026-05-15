---
title: "k3s concepts — architecture / kine / SQLite"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T09:00:00+09:00
tags: [devops, k3s, concepts]
---

# k3s concepts — architecture / kine / SQLite

**[[k3s|↑ k3s]]**

---

## 1. 아키텍처

```
[server node]                          [agent node]
  ┌────────────────────────────┐         ┌──────────────────┐
  │ k3s server (single binary) │         │ k3s agent        │
  │  ├─ kube-apiserver         │←tunnel→ │  ├─ kubelet      │
  │  ├─ scheduler              │         │  ├─ containerd   │
  │  ├─ controller-manager     │         │  ├─ kube-proxy   │
  │  ├─ kine ─→ SQLite (또는    │         │  └─ flannel      │
  │  │           etcd / DB)    │         └──────────────────┘
  │  ├─ Traefik (옵션)         │
  │  ├─ klipper-lb (옵션)      │
  │  └─ local-path (옵션)      │
  └────────────────────────────┘
```

→ k8s 의 6+ 프로세스를 **single Go binary** 로 정적 link.

---

## 2. 왜 가벼운가

### 왜 가능
- Cloud provider 통합 제거 (in-tree).
- Legacy / alpha API 제거.
- containerd 만 (Docker shim 제거).
- kine 으로 etcd 추상화 (SQLite OK).
- 정적 binary — runtime dep 적음.

### 안 줄였으면 문제
- IoT / 매장 / RPi 환경 RAM 1GB 부족.
- 부팅 시간 길어 edge 부적합.
- upgrade 다단계 (component 별).

### 트레이드오프
- 일부 alpha feature 빠짐.
- in-tree cloud provider 없음 → CCM (cloud-controller-manager) 별도.
- single binary 라 일부 deep customization 어려움.

---

## 3. kine (Kine is Not Etcd)

> kine = etcd API ↔ SQL DB adapter

```
[kube-apiserver]
     ↓ etcd v3 gRPC
[kine]
     ↓ SQL
[SQLite / Postgres / MySQL / etcd]
```

지원:
- SQLite (default, embedded)
- Postgres / MySQL / MariaDB / Microsoft SQL Server
- etcd (embedded, k3s 1.19+)
- NATS (실험)

→ etcd 운영 부담 없이 k8s.

---

## 4. 단점 (SQLite single node)

```
SQLite:
  + 0-config, 빠름 (small cluster)
  + backup 단순 (파일 1개)
  - single node 만 (HA X)
  - 큰 cluster X (500+ pod 부터 느림)
  - replicate 자체 못 함

→ HA / 큰 cluster = embedded etcd 또는 external DB.
```

---

## 5. component 별 비교

| | k8s | k3s |
| --- | --- | --- |
| **container runtime** | containerd / cri-o / Docker shim | containerd (only) |
| **CNI** | manual (Calico/Cilium/...) | flannel (VXLAN) 내장 |
| **kube-proxy** | iptables / IPVS | iptables |
| **ingress** | 별도 install | Traefik (옵션 비활성 가능) |
| **LB** | cloud / MetalLB | klipper-lb (옵션) |
| **storage** | CSI 직접 install | local-path (옵션) |
| **metrics** | 별도 install | metrics-server 내장 |
| **CCM** | cloud provider | 옵션 (--disable-cloud-controller) |

→ 모두 **옵션 disable 가능** → bare k3s 도 가능.

---

## 6. node 종류

| | 무엇 |
| --- | --- |
| **server** | control plane + worker (default 둘 다) |
| **agent** | worker only |
| **server (with --disable-agent)** | control plane only — pod 안 받음 (production 권장) |

→ production HA = 3 server (control-plane only) + N agent.

---

## 7. 데이터 저장 위치

```
/var/lib/rancher/k3s/
├── server/
│   ├── db/                   ← SQLite (state.db) 또는 etcd
│   ├── manifests/            ← auto-deploy (Traefik 등)
│   ├── tls/                  ← cert
│   └── token                 ← join token
├── agent/
│   ├── containerd/
│   ├── kubelet/
│   └── pod-manifests/
└── data/                     ← downloaded extracted binaries
```

```
/etc/rancher/k3s/
├── k3s.yaml                  ← kubeconfig (root only, 600)
├── config.yaml               ← server/agent config (★)
└── registries.yaml           ← private registry config
```

---

## 8. config.yaml (★ 추천 방식)

```yaml
# /etc/rancher/k3s/config.yaml
write-kubeconfig-mode: "0644"
tls-san:
  - k3s.example.com
  - 10.0.0.1
disable:
  - traefik
  - servicelb
node-label:
  - "role=worker"
kubelet-arg:
  - "max-pods=110"
  - "system-reserved=memory=512Mi,cpu=500m"
flannel-backend: "host-gw"      # 또는 vxlan / wireguard
```

→ command-line `--option=...` 보다 config 파일 권장 (재현성).

---

## 9. token 흐름

```
server 1 (cluster-init)
   ↓ /var/lib/rancher/k3s/server/token
   
server 2, 3 (--token=<server-token>)
   → server quorum join

agent (--token=<agent-token> 또는 server-token)
   → 그냥 worker join

server-token:  완전 권한
agent-token:   worker join 만
node-token:    각 node 의 unique join (가장 안전)
```

→ token 도난 = cluster 침투. 정기 rotation.

---

## 10. 한 줄 요약

> **k3s = single binary 화된 k8s + kine + 옵션 내장 (Traefik / klipper-lb / local-path / flannel)**

→ 옵션 끄면 vanilla k8s 와 거의 같음.

---

## 11. 관련

- [[k3s|↑ k3s]]
- [[installation]]
- [[ha-mode]]
- [[../kubernetes/concepts|↗ k8s concepts]]
