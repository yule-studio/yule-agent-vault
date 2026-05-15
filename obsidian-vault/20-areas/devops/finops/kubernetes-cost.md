---
title: "Kubernetes cost — OpenCost / Karpenter / Spot"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:10:00+09:00
tags: [devops, finops, kubernetes, opencost, karpenter]
---

# Kubernetes cost — OpenCost / Karpenter / Spot

**[[finops|↑ finops]]**

---

## 1. k8s cost 의 특수성

```
일반 EC2:
  - instance 별 비용 명확
  - tag 가 instance

k8s:
  - 1 node 위에 여러 namespace / team 의 pod
  - 사용량 추적 어려움
  - "team A 의 비용은?"
  - request vs limit vs 실사용 차이
  - overcommit / waste 흔함
```

→ 전용 도구 필요.

---

## 2. OpenCost / Kubecost (★)

```bash
# OpenCost (CNCF, OSS)
helm install opencost opencost/opencost \
    -n opencost --create-namespace

# 또는 Kubecost (UI 더 강력, free tier)
helm install kubecost kubecost/cost-analyzer \
    -n kubecost --create-namespace \
    --set kubecostToken="..."
```

기능:
- namespace / deployment / pod / label 별 비용
- 실시간 monitoring
- waste detection
- right-sizing 권장
- multi-cluster

---

## 3. cost breakdown 예 (OpenCost)

```
Namespace        | CPU req | CPU used | Memory req | Cost / day
─────────────────────────────────────────────────────────────────
prod-api         |   10    |    7     |   20Gi     | $24.50
prod-worker      |    5    |    1     |   10Gi     | $12.30
staging          |    3    |    0.5   |    8Gi     |  $7.80
dev              |    2    |    0.1   |    4Gi     |  $5.10
kube-system      |    1    |    0.5   |    2Gi     |  $3.00
```

→ "dev 가 idle 인데 $5/day = $150/mo 낭비".

---

## 4. request vs actual (★ overcommit 분석)

```
request 합:    50 CPU
actual 사용:    20 CPU
node capacity: 60 CPU

→ overcommit 30% — 절감 기회
   request 줄이거나 + autoscale
```

대시보드 metric:
```promql
# request vs usage
sum by (namespace) (kube_pod_container_resource_requests{resource="cpu"}) 
/ sum by (namespace) (rate(container_cpu_usage_seconds_total[5m]))
```

---

## 5. resource right-size (★)

```
방법:
  1. VPA (VerticalPodAutoscaler) recommendation 모드
  2. Goldilocks (Fairwinds) — UI 친화
  3. Kubecost recommendation
  4. Datadog APM 분석
  5. Prometheus query 직접

# Goldilocks
kubectl create namespace goldilocks
helm install goldilocks fairwinds-stable/goldilocks -n goldilocks

# namespace 에 enable
kubectl label namespace prod-api goldilocks.fairwinds.com/enabled=true
```

→ "이 pod 는 request 1000m 인데 권장 200m" 보여줌.

---

## 6. Karpenter (★ 표준)

```yaml
# 다양 instance + Spot 자동 선택
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata: {name: default}
spec:
  template:
    spec:
      requirements:
        - {key: kubernetes.io/arch, operator: In, values: [amd64, arm64]}
        - {key: karpenter.sh/capacity-type, operator: In, values: [spot, on-demand]}
        - {key: karpenter.k8s.aws/instance-category, operator: In, values: [c, m, r]}
        - {key: karpenter.k8s.aws/instance-size, operator: NotIn, values: [nano, micro]}
      nodeClassRef: {name: default}

  disruption:
    consolidationPolicy: WhenUnderutilized      # ★ 자주 consolidate
    consolidateAfter: 30s
    expireAfter: 720h                            # 30일마다 새로

  limits:
    cpu: 1000

---
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata: {name: default}
spec:
  amiFamily: Bottlerocket
  subnetSelectorTerms:
    - tags: {karpenter.sh/discovery: "my-cluster"}
  securityGroupSelectorTerms:
    - tags: {karpenter.sh/discovery: "my-cluster"}
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs: {volumeSize: 50Gi, volumeType: gp3}
```

→ 30-50% node cost 절감 흔함.

---

## 7. consolidation (★)

```
일반:
  20 node, 50% 만 사용 → 그냥 둠 (cost ↓ X)

Karpenter consolidation:
  20 node 중 가능한 pod 들 → 더 적은 node 로 packing
  여유 node 자동 termination
  
→ underutilization 자동 해결.
```

---

## 8. PDB + Karpenter

```yaml
# PDB 가 graceful 보장
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: {name: api}
spec:
  minAvailable: 80%
  selector: {matchLabels: {app: api}}
```

→ Karpenter consolidation 시 PDB 존중. service 끊김 X.

---

## 9. KEDA — scale to 0 (★)

```yaml
# 사용 없을 때 0 pod
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata: {name: worker}
spec:
  scaleTargetRef: {name: worker}
  minReplicaCount: 0         # ★ 0
  maxReplicaCount: 100
  pollingInterval: 30
  cooldownPeriod: 300
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka:9092
        consumerGroup: workers
        topic: jobs
        lagThreshold: "10"
```

→ idle 시 비용 0.

---

## 10. namespace 격리 + ResourceQuota

```yaml
# namespace 의 max 자원 제한
apiVersion: v1
kind: ResourceQuota
metadata: {name: dev-quota, namespace: dev}
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    persistentvolumeclaims: "5"
    pods: "20"
```

→ team / env 별 비용 cap.

---

## 11. namespace LimitRange (default 강제)

```yaml
apiVersion: v1
kind: LimitRange
metadata: {name: defaults}
spec:
  limits:
    - type: Container
      default: {cpu: 200m, memory: 256Mi}        # default limit
      defaultRequest: {cpu: 100m, memory: 128Mi}  # default request
      max: {cpu: "4", memory: 8Gi}                # 최대 허용
      min: {cpu: 10m, memory: 16Mi}               # 최소
```

→ request / limit 명시 안 한 pod 도 안전.

---

## 12. cluster autoscaler 대안

```
Cluster Autoscaler:
  - ASG / node group 기반
  - 단순 / 안정
  - Karpenter 대체로 사용

Karpenter:
  - 더 빠름 (수십초 vs 분)
  - 다양 instance type / Spot
  - consolidation
  - bin-packing
  - AWS 만 (현재)

GKE Autopilot:
  - serverless k8s
  - GKE 가 모든 node 관리
  - Karpenter 비슷
```

---

## 13. ARM 활용 (k8s)

```
ARM64 노드 (Graviton):
  - 20% 저렴
  - 15-30% 성능

migration:
  1. container multi-arch build
     docker buildx build --platform linux/amd64,linux/arm64 --push

  2. Karpenter NodePool 의 arch 다중
     requirements:
       - {key: arch, operator: In, values: [amd64, arm64]}

  3. Pod spec nodeSelector 안 함 (Karpenter 자동)
```

---

## 14. logs / monitoring cost

```
Datadog: $/host/mo + log ingest
  → 비싼 host 줄임 (sample / aggregate)

Prometheus + Mimir / Thanos:
  - 자체 host = cheap
  - metric 의 cardinality 제한 (높으면 비쌈)

trace (Tempo / Jaeger):
  - sampling 1%
  - object storage 저렴
```

---

## 15. namespace 별 cost dashboard

```
Grafana 의 OpenCost panel:
  Top 10 most expensive namespace
  CPU/memory/storage breakdown
  request vs actual
  Spot vs on-demand
  idle resource ($ wasted)

→ 매주 review.
```

---

## 16. 함정

1. **request = limit 항상** — overcommit 불가, 비쌈.
2. **request 너무 큼** — node 더 많이 필요.
3. **LimitRange 없음** — request 안 적은 pod = 무제한.
4. **PDB 없음 + consolidation** — 한꺼번 종료.
5. **Karpenter + critical workload spot only** — capacity fail.
6. **monitoring 자체가 비쌈** — Prometheus cardinality 폭주.
7. **dev 의 over-spec** — prod 와 동일 size.

---

## 17. 관련

- [[finops|↑ finops]]
- [[right-sizing]]
- [[reserved-and-spot]]
- [[../kubernetes/autoscaling|↗ k8s autoscale]]
- [[../kubernetes/concepts|↗ k8s]]
