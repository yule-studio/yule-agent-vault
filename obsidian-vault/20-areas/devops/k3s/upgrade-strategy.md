---
title: "k3s upgrade — channel / 자동 upgrade controller"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T09:11:00+09:00
tags: [devops, k3s, upgrade]
---

# k3s upgrade — channel / 자동 upgrade controller

**[[k3s|↑ k3s]]**

---

## 1. version 정책

```
k3s tag = <k8s version>+k3s<revision>

예: v1.29.4+k3s1 = k8s 1.29.4 + k3s patch 1
```

→ k8s upstream 따라 매 minor 마다 새 channel.

---

## 2. channel

| channel | 의미 |
| --- | --- |
| **stable** (default) | 안정 (몇 주 검증 후) |
| **latest** | 최신 |
| **testing** | 다음 release 후보 |
| **v1.29** | 특정 minor 의 latest patch |
| **v1.29.4+k3s1** | 정확한 version 고정 |

→ **production = 특정 version 고정**. `stable` 도 자동 업데이트되니 신중.

---

## 3. manual upgrade (single node)

```bash
# server
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.29.4+k3s1 sh -

# 자동:
# 1. binary 다운
# 2. k3s service restart
# 3. control plane 재시작
# 4. pod 재시작 (kubelet)

# 검증
kubectl version
kubectl get nodes
```

→ 한 명령. systemd 자동 restart. 5분 안에.

---

## 4. HA cluster upgrade 순서 (★)

```
3 server cluster:

Step 1. drain server 1
  kubectl drain server1 --ignore-daemonsets --delete-emptydir-data

Step 2. upgrade server 1
  ssh server1
  curl ... INSTALL_K3S_VERSION=... sh -

Step 3. uncordon
  kubectl uncordon server1

Step 4. 5분 대기 — etcd 동기화

Step 5. server 2, 3 반복

Step 6. agent 들 upgrade
  for agent in agent1 agent2 ...; do
      kubectl drain $agent --ignore-daemonsets
      ssh $agent "curl ... sh -"
      kubectl uncordon $agent
  done
```

→ 한 번에 하나씩. quorum 유지.

---

## 5. system-upgrade-controller (★ 자동)

```bash
# 설치
kubectl apply -f https://github.com/rancher/system-upgrade-controller/releases/latest/download/system-upgrade-controller.yaml
```

```yaml
# Plan — server upgrade
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: server-upgrade
  namespace: system-upgrade
spec:
  concurrency: 1                # 한 번에 하나
  cordon: true
  nodeSelector:
    matchExpressions:
      - {key: node-role.kubernetes.io/control-plane, operator: Exists}
  serviceAccountName: system-upgrade
  upgrade:
    image: rancher/k3s-upgrade
  version: v1.29.4+k3s1
```

```yaml
# Plan — agent upgrade
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata: {name: agent-upgrade, namespace: system-upgrade}
spec:
  concurrency: 2
  cordon: true
  nodeSelector:
    matchExpressions:
      - {key: node-role.kubernetes.io/control-plane, operator: DoesNotExist}
  prepare:
    image: rancher/k3s-upgrade
    args: ["prepare", "server-upgrade"]    # server 먼저 끝나길 wait
  serviceAccountName: system-upgrade
  upgrade:
    image: rancher/k3s-upgrade
  version: v1.29.4+k3s1
```

→ apply → controller 가 자동 drain / upgrade / uncordon.

---

## 6. label 기반 partial upgrade

```yaml
spec:
  nodeSelector:
    matchExpressions:
      - {key: canary, operator: Exists}    # canary label 있는 node 만
```

→ 일부 canary node 먼저 upgrade → 모니터 → 확대.

---

## 7. rollback

```bash
# k3s 자체는 rollback 명령 없음 — 다시 옛 version 설치

# 1. 옛 version 결정
kubectl get nodes -o wide | head

# 2. install
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.29.3+k3s1 sh -
```

→ 단, etcd 의 신 version 데이터 변경된 부분이 옛 version 에서 안 읽힐 수 있음. **etcd snapshot** 필수.

---

## 8. upgrade 전 점검

```
☐ 새 version 의 changelog 검토
  - breaking change?
  - deprecated API?

☐ k8s API 호환성
  - kubectl convert (또는 pluto)
  - 사용 중인 CRD 호환?

☐ etcd snapshot (HA 시)
  sudo k3s etcd-snapshot save

☐ external DB backup (external DB 시)
  pg_dump ...

☐ 현재 manifests 확인
  ls /var/lib/rancher/k3s/server/manifests/

☐ resource 변동 (cluster-cidr 등) 없는지
☐ disk 여유 (20%+)
☐ 비peak time
☐ runbook 준비 (rollback 절차)
```

---

## 9. minor version skip 금지

```
✓ 1.28 → 1.29 (one minor)
✗ 1.27 → 1.29 (두 minor — 위험)
```

→ k8s 정책 — minor 하나씩. version skew skew policy.

---

## 10. component compatibility

```
kubectl: 새 / 옛 1 minor 차이 OK
kubelet: control plane 보다 1 minor 낮 OK
controller / scheduler: 정확히 같은 minor
```

→ control plane 먼저, 그 다음 worker.

---

## 11. CRD 마이그레이션 (★)

```
일부 minor 에서 alpha → beta → GA
  v1alpha1 → v1beta1 → v1

예: networking.k8s.io/v1beta1 Ingress (1.18) → networking.k8s.io/v1 (1.22)

방법:
  1. 새 version 검토
  2. kubectl convert 또는 pluto 로 사용 중 API 검색
  3. manifest 업데이트
  4. apply
  5. 그 후 cluster upgrade
```

```bash
# pluto — deprecated API 검색
pluto detect-helm --target-versions=k8s=v1.29

# kubectl convert (deprecated 도구)
kubectl convert -f old.yaml --output-version=networking.k8s.io/v1
```

---

## 12. 함정

1. **`stable` channel 자동 upgrade** — 의도치 않은 minor jump.
2. **agent 가 먼저 upgrade** — control plane 보다 신 version → error.
3. **etcd snapshot 안 함** — rollback 못 함.
4. **minor 2개 jump** — k8s 정책 위반.
5. **CRD migration 안 함** — apply 실패.
6. **off-peak 안 함** — 사용자 영향.
7. **controller 의 plan 두 번 실행** — 동시 drain → cluster down.

---

## 13. 관련

- [[k3s|↑ k3s]]
- [[ha-mode]]
- [[backup-restore]]
- [[production-checklist]]
