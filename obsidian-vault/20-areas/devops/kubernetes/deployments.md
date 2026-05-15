---
title: "Deployments — rolling / blue-green / canary ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T03:42:00+09:00
tags: [devops, kubernetes, deployment]
---

# Deployments — rolling / blue-green / canary ★

**[[kubernetes|↑ k8s]]**

---

## 1. 3 가지 배포 전략

| 전략 | 동작 | 비용 |
| --- | --- | --- |
| **Rolling** ★ | N 개 씩 점진 교체 | 0 (기본) |
| **Blue-Green** | new (green) 띄우고 한 번에 switch | 2배 인스턴스 |
| **Canary** | 일부 (5%) → 점진 (100%) | 약간 + 모니터링 필요 |
| Recreate | 모두 죽이고 새로 | downtime |

---

## 2. Rolling (기본)

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%        # 한 번에 추가 생성 max
      maxUnavailable: 25%  # 한 번에 죽을 max
```

```bash
kubectl set image deploy/web app=myapp:1.1
kubectl rollout status deploy/web
kubectl rollout undo deploy/web   # 롤백
```

---

## 3. Blue-Green

```yaml
# blue (v1.0)
apiVersion: apps/v1
kind: Deployment
metadata: {name: web-blue}
spec:
  selector: {matchLabels: {app: web, version: blue}}
  ...

# green (v1.1) — 새로 띄움
apiVersion: apps/v1
kind: Deployment
metadata: {name: web-green}
spec:
  selector: {matchLabels: {app: web, version: green}}
  ...

# Service — selector 만 변경 시 전환
apiVersion: v1
kind: Service
metadata: {name: web}
spec:
  selector: {app: web, version: green}   # blue → green
```

→ rollback 즉시 (selector 만 되돌림).

---

## 4. Canary (Argo Rollouts / Flagger)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata: {name: web}
spec:
  strategy:
    canary:
      steps:
        - setWeight: 5
        - pause: {duration: 5m}
        - setWeight: 25
        - pause: {duration: 10m}
        - setWeight: 50
        - pause: {duration: 10m}
        - setWeight: 100
      analysis:
        templates:
          - templateName: success-rate
        startingStep: 2
```

→ metric (success rate) 기반 자동 promote / rollback.

---

## 5. Probes (필수)

```yaml
livenessProbe:
  httpGet: {path: /actuator/health/liveness, port: 8080}
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet: {path: /actuator/health/readiness, port: 8080}
  initialDelaySeconds: 10
  periodSeconds: 5

startupProbe:
  httpGet: {path: /actuator/health, port: 8080}
  failureThreshold: 30
  periodSeconds: 10
```

| Probe | 의미 |
| --- | --- |
| **liveness** | 살아있는지 (fail → 재시작) |
| **readiness** | traffic 받을 수 있는지 (fail → service 에서 빠짐) |
| **startup** | 시작 중 (다른 probe 비활성) — JVM warmup 60s 같은 케이스 |

---

## 6. resource requests / limits

```yaml
resources:
  requests:                # 노드 scheduling 기준
    cpu: 100m              # 0.1 vCPU
    memory: 256Mi
  limits:                  # 절대 한도
    cpu: 500m
    memory: 512Mi
```

→ JVM 의 경우 limits.memory 와 `-XX:MaxRAMPercentage=75` 함께.

---

## 7. PodDisruptionBudget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: {name: web}
spec:
  minAvailable: 2
  selector: {matchLabels: {app: web}}
```

→ rolling / node drain 시 항상 2 Pod 유지.

---

## 8. 함정

1. **readiness probe 없음** → traffic 이 시작 안 된 Pod 에.
2. **probe 가 너무 strict** → 정상 Pod 도 죽음.
3. **resource limit 없음** → 노드 OOM.
4. **maxUnavailable=0 + replica 1** → 배포 불가능.
5. **PDB 없음** → rolling + node drain 동시 → 0 replica.

---

## 9. 관련

- [[kubernetes|↑ k8s]]
- [[concepts]]
- [[autoscaling]]
