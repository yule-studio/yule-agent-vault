---
title: "Autoscaling — HPA / VPA / Cluster Autoscaler / KEDA"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T03:54:00+09:00
tags: [devops, kubernetes, autoscaling, hpa]
---

# Autoscaling — HPA / VPA / Cluster Autoscaler / KEDA

**[[kubernetes|↑ k8s]]**

---

## 1. 종류

| | 무엇 | 사용 |
| --- | --- | --- |
| **HPA** (Horizontal) ★ | Pod replica 늘림 | 일반 |
| **VPA** (Vertical) | Pod resources 늘림 | 변동 적은 워크로드 |
| **Cluster Autoscaler** | 노드 늘림 | 둘 다 필요 |
| **KEDA** | event-driven (Kafka lag / SQS queue) | 큐 기반 |

---

## 2. HPA (CPU 기반)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: {name: web}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 3
  maxReplicas: 30
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100        # 한 번에 2배
          periodSeconds: 60
```

---

## 3. HPA (custom metric — Prometheus)

```yaml
metrics:
  - type: Pods
    pods:
      metric: {name: http_requests_per_second}
      target:
        type: AverageValue
        averageValue: "100"
```

→ prometheus-adapter 필요.

---

## 4. KEDA (Kafka lag)

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata: {name: consumer}
spec:
  scaleTargetRef: {name: kafka-consumer}
  minReplicaCount: 1
  maxReplicaCount: 30
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka:9092
        consumerGroup: my-group
        topic: events
        lagThreshold: "100"
```

→ event-driven (Kafka / SQS / RabbitMQ / Redis) 기반 scale.

---

## 5. Cluster Autoscaler (노드)

```bash
# EKS — Managed Node Group + autoscaler add-on
# GKE — Cluster Autoscaler 내장
# AKS — Cluster Autoscaler 통합
```

---

## 6. 함정

1. **requests 누락** → HPA 동작 X (utilization 계산 불가).
2. **scaleUp 너무 빠름** → flapping.
3. **scaleDown 너무 빠름** → cold start 폭주.
4. **min=0** → 첫 요청 cold start.
5. **노드 add 시간** (cloud) — 3분+ → 미리 늘림.

---

## 7. 관련

- [[kubernetes|↑ k8s]]
- [[deployments]]
- [[../monitoring/monitoring|↗ monitoring]]
