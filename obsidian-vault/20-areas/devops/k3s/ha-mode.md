---
title: "k3s HA — embedded etcd / external DB"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T09:05:00+09:00
tags: [devops, k3s, ha, etcd]
---

# k3s HA — embedded etcd / external DB

**[[k3s|↑ k3s]]**

---

## 1. HA 가 왜 필요

```
single server:
  server down → cluster unavailable (deploy / scale 못 함)
  data corruption → 전체 손실

HA (3 server):
  1 server down → 자동 운영
  rolling upgrade 가능
  backup quorum
```

→ production = HA 필수.

---

## 2. 3 가지 HA 모델

| 모델 | 장 | 단 |
| --- | --- | --- |
| **Embedded etcd** | self-contained, 3 server, 운영 단순 | 3 server 정확, network low-latency 필요 |
| **External DB** (Postgres/MySQL) | DB backup 친화, scale-out 쉬움 | DB 자체 HA 부담 |
| **External etcd** | k8s 표준, 큰 cluster | 운영 부담 |

→ 일반 = **embedded etcd**. DB 인프라 있으면 external DB.

---

## 3. embedded etcd 설치

### 첫 server (cluster-init)

```yaml
# /etc/rancher/k3s/config.yaml (server 1)
token: "my-cluster-shared-secret"
cluster-init: true                # ★ 새 cluster 의 첫 server
tls-san:
  - k3s.example.com               # LB 도메인
  - 10.0.0.10                     # LB IP
write-kubeconfig-mode: "0644"
node-taint:
  - "CriticalAddonsOnly=true:NoExecute"   # server 에 일반 pod 안 받음 (★ production)
```

```bash
curl -sfL https://get.k3s.io | sh -
```

### 추가 server (join)

```yaml
# /etc/rancher/k3s/config.yaml (server 2, 3)
token: "my-cluster-shared-secret"
server: "https://server1.example.com:6443"
tls-san:
  - k3s.example.com
node-taint:
  - "CriticalAddonsOnly=true:NoExecute"
```

```bash
curl -sfL https://get.k3s.io | sh -
```

### agent (worker)

```yaml
# /etc/rancher/k3s/config.yaml (agent)
token: "my-cluster-shared-secret"
server: "https://k3s.example.com:6443"   # ★ LB 사용
```

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://k3s.example.com:6443 K3S_TOKEN=xxx sh -
```

---

## 4. 왜 etcd 는 3, 5, 7 (홀수)

```
quorum = (N/2) + 1

N=2: quorum=2 — 둘 다 살아야 → split-brain 불가하지만 1 fail = down
N=3: quorum=2 — 1 fail OK
N=5: quorum=3 — 2 fail OK
N=7: quorum=4 — 3 fail OK

짝수 (4) = 홀수 (3) 와 동일 fault tolerance 인데 더 비싸.
```

→ **3 server** 가 비용/안정 균형. 매우 큰 = 5.

---

## 5. external DB (Postgres) 설치

```bash
# DB 미리 준비 (managed RDS / Cloud SQL 권장)
# postgres://k3s:secret@db.internal:5432/k3s?sslmode=require

# server 1
curl -sfL https://get.k3s.io | sh -s - server \
    --datastore-endpoint="postgres://k3s:secret@db.internal:5432/k3s?sslmode=require" \
    --token="my-cluster-secret" \
    --tls-san k3s.example.com

# server 2, 3
curl -sfL https://get.k3s.io | sh -s - server \
    --datastore-endpoint="postgres://k3s:secret@db.internal:5432/k3s?sslmode=require" \
    --token="my-cluster-secret"
```

→ embedded etcd 와 달리 `cluster-init` 안 함. 모든 server 가 같은 DB 가리킴.

→ DB 의 HA = managed (RDS Multi-AZ) 권장.

---

## 6. embedded etcd vs external DB

| | embedded etcd | external DB |
| --- | --- | --- |
| 운영 부담 | 낮음 (k3s 가 관리) | 높음 (DB HA / backup 별도) |
| latency | 낮음 (local) | 네트워크 |
| scale | 3-7 server | DB 의 한계 |
| backup | `k3s etcd-snapshot` 명령 | DB native (pg_dump) |
| 마이그레이션 | 한 번 결정하면 변경 어려움 | 쉬움 |
| 비용 | server cost | + DB cost |
| 권장 | 일반 | 이미 DB 인프라 있을 때 |

---

## 7. load balancer (★ 필수)

```
[client / agent]
       ↓
[LB (k3s.example.com:6443)]
   │      │      │
   ↓      ↓      ↓
server1 server2 server3
```

LB 선택:
- **HAProxy** (간단, OSS)
- **kube-vip** (k8s 안에 LB)
- **MetalLB** (자체 LB)
- **AWS NLB / GCP TCP LB** (cloud)
- **DNS round-robin** (단순, fail-over 느림)

```
# HAProxy 예
backend k3s_apiserver
    mode tcp
    balance roundrobin
    option tcp-check
    server s1 server1:6443 check
    server s2 server2:6443 check
    server s3 server3:6443 check

frontend k3s_apiserver
    bind *:6443
    mode tcp
    default_backend k3s_apiserver
```

---

## 8. server 의 NoSchedule taint (★ production)

```yaml
node-taint:
  - "CriticalAddonsOnly=true:NoExecute"
```

→ server 노드는 control plane 만. 일반 worker pod 안 받음.  
→ 안 하면 control plane 과 app 이 같은 노드 → app OOM 시 cluster down.

---

## 9. 인증서 갱신

```bash
# 자동 — k3s 가 cert 90일 이내 만료 시 startup 시 자동 갱신
sudo systemctl restart k3s

# 강제
sudo k3s certificate rotate --service apiserver
sudo k3s certificate rotate --service kubelet
sudo systemctl restart k3s
```

→ HA 에서 한 server 씩 rotation. 한 번에 모두 X.

---

## 10. node 추가 / 제거

```bash
# 추가
curl -sfL https://get.k3s.io | K3S_URL=... K3S_TOKEN=... sh -

# 제거 (server)
kubectl drain node-name --ignore-daemonsets --delete-emptydir-data
# embedded etcd 의 경우
sudo k3s etcd-snapshot save    # 안전을 위해
sudo k3s server --cluster-reset        # ★ 마지막 server 또는 quorum 회복 시

# 제거 (agent)
kubectl drain node-name
kubectl delete node node-name
sudo /usr/local/bin/k3s-agent-uninstall.sh
```

---

## 11. quorum 잃음 (★ 위기 복구)

```
2 server down (3 중) → quorum 잃음 → cluster 멈춤

복구 옵션:
  A. 잃은 server 복구
  B. 마지막 server 에서 cluster-reset
       sudo systemctl stop k3s
       sudo k3s server --cluster-reset
       # → single-node mode 로 회복
       # → 새 server 들 join
```

→ 정기 **etcd snapshot** 으로 backup. [[backup-restore]] 참조.

---

## 12. 함정

1. **2 server** — quorum 2, 1 fail 시 down. 3 또는 5.
2. **server 에 일반 pod** — taint 없으면 control plane 영향.
3. **LB 없이 첫 server IP 사용** — 그 server down 시 join 못 함.
4. **cluster-init 두 번** — 두 다른 cluster 가 됨.
5. **token 다름** — join 실패.
6. **external DB 의 single instance** — DB down = cluster 마비.
7. **etcd disk slow** — fsync latency → cluster 느림. SSD 권장.

---

## 13. 관련

- [[k3s|↑ k3s]]
- [[installation]]
- [[backup-restore]]
- [[../kubernetes/managed-vs-self-hosted|↗ managed k8s]]
