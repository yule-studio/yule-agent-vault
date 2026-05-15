---
title: "Advanced scheduling — affinity / taint / priority / topology"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:24:00+09:00
tags: [devops, kubernetes, scheduling]
---

# Advanced scheduling — affinity / taint / priority / topology

**[[kubernetes|↑ kubernetes]]**

---

## 1. 기본 scheduler 흐름

```
1. Filtering (predicate)
   - node 의 resource 충분?
   - nodeSelector 매칭?
   - taint / toleration?
   - port 충돌?

2. Scoring (priority)
   - 가장 좋은 node 선택
   - resource balance
   - image locality
   - affinity 등

3. Binding
   - pod → node
```

---

## 2. nodeSelector (단순)

```yaml
spec:
  nodeSelector:
    disktype: ssd
    gpu: "true"
```

→ 단순 (모든 조건 AND). 더 복잡 = affinity.

---

## 3. nodeAffinity (★)

```yaml
spec:
  affinity:
    nodeAffinity:
      # 필수 (안 맞으면 스케줄 X)
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values: [ssd, nvme]
              - key: kubernetes.io/arch
                operator: In
                values: [arm64]
      
      # 선호 (안 맞으면 다른 node 도 OK)
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          preference:
            matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values: [ap-northeast-2a]
```

→ "required" + "preferred" mix. 일반 / fallback.

---

## 4. podAffinity / podAntiAffinity (★)

```yaml
# 같이 (cache 와 app)
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels: {app: redis}
          topologyKey: kubernetes.io/hostname
```

→ "redis pod 와 같은 node 에 schedule" (네트워크 latency ↓).

```yaml
# 분리 (replica 분산)
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels: {app: web}
          topologyKey: topology.kubernetes.io/zone
```

→ "같은 zone 에 같은 app pod 2개 X" → multi-AZ 강제.

---

## 5. taint / toleration (★)

```bash
# node 에 taint (이 node 는 special)
kubectl taint nodes gpu-node-1 dedicated=gpu:NoSchedule

# 의미:
# 일반 pod 는 gpu-node-1 에 schedule 안 됨
# (toleration 가진 pod 만 가능)
```

```yaml
spec:
  tolerations:
    - key: dedicated
      operator: Equal
      value: gpu
      effect: NoSchedule
```

```
taint effect 3종:
  NoSchedule        — toleration 없으면 schedule X
  PreferNoSchedule  — 가능하면 안 함 (soft)
  NoExecute         — 이미 있는 pod 도 추출
```

→ 흔한 사용:
- master node (system pod 만)
- GPU node
- Spot node (graceful taint by termination handler)
- 특정 workload 격리

---

## 6. topologySpreadConstraints (★ 권장)

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels: {app: web}
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: ScheduleAnyway
      labelSelector:
        matchLabels: {app: web}
```

→ "zone 별로 +- 1 안에 균등 + node 별 가능하면 분산".

→ podAntiAffinity 보다 더 유연.

---

## 7. priorityClass (★)

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata: {name: high-priority}
value: 1000000
globalDefault: false
description: "production critical workload"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata: {name: best-effort}
value: 0
```

```yaml
spec:
  priorityClassName: high-priority
```

→ resource 부족 시 priority 높은 게 다른 pod evict.

→ system-cluster-critical, system-node-critical 은 default 매우 높음.

---

## 8. resource requests / limits

```yaml
spec:
  containers:
    - name: web
      resources:
        requests:           # scheduler 가 보는 값
          cpu: 100m
          memory: 128Mi
        limits:             # OOM kill 한계
          cpu: 500m
          memory: 512Mi
```

```
QoS class (자동):
  - Guaranteed   : request == limit (모든 resource)
  - Burstable    : request < limit
  - BestEffort   : 둘 다 없음

OOM 시 eviction 순서:
  BestEffort → Burstable → Guaranteed (마지막)
```

→ critical = Guaranteed.

---

## 9. DaemonSet (모든 node 에 1개)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata: {name: log-collector}
spec:
  selector: {matchLabels: {app: log}}
  template:
    metadata: {labels: {app: log}}
    spec:
      hostNetwork: true            # 자주 사용
      hostPID: true
      tolerations:
        - operator: Exists          # 모든 taint 무시 (master 도)
      containers:
        - name: collector
          image: fluentbit:latest
```

→ log collector / monitoring agent / network plugin / CSI driver.

---

## 10. PDB (PodDisruptionBudget) (★)

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: {name: web-pdb}
spec:
  minAvailable: 2              # 또는 70%
  selector: {matchLabels: {app: web}}
```

→ node drain / Karpenter consolidation / autoscaler 시 PDB 존중.

→ 한 번에 너무 많이 종료 X.

---

## 11. custom scheduler

```
default scheduler 만으로 부족 시 자체 작성.

용도:
  - GPU job scheduling (gang scheduling)
  - ML workload (batch)
  - 특수 hardware
  - cost-aware scheduling

도구:
  - Volcano (CNCF, batch / AI)
  - YuniKorn (Apache, multi-tenant)
  - Karpenter (autoscale + scheduling)
  - 자체 (scheduler framework)
```

```yaml
# 특정 scheduler 사용
spec:
  schedulerName: volcano
```

---

## 12. descheduler (★)

```
default scheduler 는 한 번 schedule 하면 안 옮김.
시간 지나면 imbalanced (한 node 가 hot).

Descheduler 가 정기 evict → 재schedule:
  - LowNodeUtilization
  - RemoveDuplicates
  - RemovePodsViolatingNodeAffinity
  - RemovePodsViolatingTopologySpreadConstraint

cron / deployment 으로 실행.
```

---

## 13. 흔한 패턴 예

### dedicated GPU node

```yaml
# Node label + taint:
kubectl label node gpu-1 hardware=gpu
kubectl taint node gpu-1 hardware=gpu:NoSchedule

# Pod 의 spec:
spec:
  nodeSelector:
    hardware: gpu
  tolerations:
    - key: hardware
      operator: Equal
      value: gpu
      effect: NoSchedule
  containers:
    - resources:
        limits:
          nvidia.com/gpu: 1
```

### multi-AZ 분산

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels: {app: web}
```

### preempt for critical

```yaml
# 일반 batch
spec:
  priorityClassName: low

# Critical web
spec:
  priorityClassName: high       # batch evict 후 schedule
```

### Spot + on-demand mix

```yaml
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 70
          preference:
            matchExpressions:
              - key: karpenter.sh/capacity-type
                operator: In
                values: [spot]
        - weight: 30
          preference:
            matchExpressions:
              - key: karpenter.sh/capacity-type
                operator: In
                values: [on-demand]
```

---

## 14. 함정

1. **podAntiAffinity required + node 부족** — pending.
2. **taint without toleration** — 의도 외 어디에도 schedule X.
3. **PDB 없음** — drain 시 service down.
4. **priorityClass 없음 + critical** — evict 됨.
5. **topology key 오타** — 의도와 다름.
6. **DaemonSet 의 toleration 없음** — master 에 안 깜.
7. **Guaranteed = waste** — request = limit → bin-pack 못 함.

---

## 15. 관련

- [[kubernetes|↑ kubernetes]]
- [[autoscaling]]
- [[concepts]]
- [[../finops/kubernetes-cost|↗ k8s cost]]
