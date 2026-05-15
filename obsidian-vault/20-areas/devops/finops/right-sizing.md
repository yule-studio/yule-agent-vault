---
title: "Right-sizing — 적정 자원 + autoscaling"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:04:00+09:00
tags: [devops, finops, right-sizing]
---

# Right-sizing — 적정 자원 + autoscaling

**[[finops|↑ finops]]**

---

## 1. 왜

```
초기 디자인:
  "안전하게 큰 instance" — over-provision

실제:
  - 평균 CPU 10% 만 사용
  - 메모리 30% 만
  - 큰 instance → 평균 utilization 매우 낮음

right-sizing:
  - 실측 → 적정 size
  - 50% 비용 절감 흔함
```

---

## 2. 분석 도구

```
CloudWatch / Datadog:
  - CPU avg / max
  - memory avg / max
  - network in / out
  - disk IO

AWS Compute Optimizer:
  - 14일 분석
  - "이 instance 는 t3.large → t3.small 권장"
  - 자동 권장

Trusted Advisor:
  - Low utilization (idle resource)
  - free tier

VMware / k8s VPA:
  - VerticalPodAutoscaler 가 권장
```

---

## 3. 결정 매트릭스

```
CPU avg < 5% + max < 20%:
  → 1-2 size 작게

CPU avg < 20% + max < 60%:
  → 1 size 작게 (또는 동일, autoscale 추가)

CPU avg > 80%:
  → 1 size 크게 또는 scale-out

memory: 마찬가지

network bound:
  → 더 큰 network 의 instance family (m5n, c5n)
```

→ 평균만 보지 말고 p95 / p99 도 검토.

---

## 4. instance family 선택

| family | 무엇 | 사용 |
| --- | --- | --- |
| **t** (burstable) | CPU credit | low / sporadic CPU |
| **m** (general) | balanced | 일반 |
| **c** (compute) | CPU 강 | encoding / batch |
| **r** (memory) | memory 강 | cache / Java heap |
| **i** (storage) | NVMe SSD local | DB |
| **g** (GPU) | NVIDIA | ML / video |
| **inf / trn** | AI 전용 | ML inference / training |

→ workload type 별 선택. m5.large + 메모리 부족 = r5.large 가 더 저렴할 수 있음.

---

## 5. ARM (Graviton) 활용 (★)

```
AWS Graviton2 / 3:
  - x86 보다 20% 저렴
  - 15-30% 성능 ↑
  - container / Java / Python 거의 호환

migration:
  - container image 의 multi-arch (linux/arm64)
  - benchmark
  - 점진 전환

c7g / m7g / r7g family.
```

→ **대부분 workload 에 즉시 도입 가능**.

---

## 6. autoscaling 패턴

### A. ASG (EC2)

```hcl
resource "aws_autoscaling_group" "web" {
  min_size         = 2
  max_size         = 20
  desired_capacity = 5

  target_group_arns = [aws_lb_target_group.web.arn]
}

# CPU 기반 scale
resource "aws_autoscaling_policy" "cpu" {
  autoscaling_group_name = aws_autoscaling_group.web.name
  policy_type            = "TargetTrackingScaling"
  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value = 60.0   # 60% 목표
  }
}
```

### B. k8s HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: {name: web}
spec:
  scaleTargetRef: {kind: Deployment, name: web}
  minReplicas: 3
  maxReplicas: 50
  metrics:
    - type: Resource
      resource: {name: cpu, target: {type: Utilization, averageUtilization: 70}}
    - type: Resource
      resource: {name: memory, target: {type: Utilization, averageUtilization: 80}}
    # custom metric (RPS)
    - type: Pods
      pods:
        metric: {name: http_requests_per_second}
        target: {type: AverageValue, averageValue: "100"}
```

### C. KEDA (★ event-driven)

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata: {name: worker}
spec:
  scaleTargetRef: {name: worker}
  minReplicaCount: 0           # ★ 0 가능! (no traffic = no cost)
  maxReplicaCount: 100
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka:9092
        consumerGroup: workers
        topic: jobs
        lagThreshold: "10"     # 10 message 마다 1 pod
```

→ Kafka lag / SQS depth / cron 등으로 scale. 0 까지.

---

## 7. predictive autoscaling

```
일반 reactive:
  load 증가 → metric 감지 → scale → 약간 늦음

predictive:
  ML 이 trend 예측 → 미리 scale
  
도구:
  - AWS Predictive Scaling
  - GCP Recommender
  - Datadog Predictive Forecast
```

→ daily / weekly 패턴이 있을 때 유리.

---

## 8. dev / staging 끄기 (★ 큰 절감)

```
prod 24/7 = $$$ (피할 수 없음)
dev / staging 24/7 = $$$ (낭비)

방법:
  - 매 평일 19시 → stop
  - 매 평일 09시 → start
  - 주말 stop
  - 약 60-70% 절감

도구:
  - AWS Instance Scheduler
  - Lambda + EventBridge
  - k8s CronJob 으로 deployment scale 0
```

```yaml
# CronJob 예
apiVersion: batch/v1
kind: CronJob
metadata: {name: stop-dev-evening}
spec:
  schedule: "0 19 * * 1-5"     # 평일 19시
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: kubectl
              image: bitnami/kubectl
              command:
                - kubectl
                - scale
                - --replicas=0
                - deployment
                - --all
                - -n
                - dev
```

---

## 9. cluster autoscaler / Karpenter

```
Cluster Autoscaler:
  - pod 가 unschedulable → node 추가
  - 사용 안 하는 node → 삭제
  - 단점: ASG / node group 미리 정의

Karpenter (★ AWS 권장):
  - 더 빠름 (수십초)
  - 다양한 instance type 자동 선택
  - Spot 친화
  - bin-packing 좋음
```

```yaml
# Karpenter NodePool
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata: {name: default}
spec:
  template:
    spec:
      requirements:
        - {key: kubernetes.io/arch, operator: In, values: [amd64, arm64]}
        - {key: karpenter.sh/capacity-type, operator: In, values: [spot, on-demand]}
        - {key: node.kubernetes.io/instance-type, operator: In, values: [t3.medium, t3.large, m5.large]}
      nodeClassRef: {name: default}
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 720h
  limits:
    cpu: 1000
```

---

## 10. database right-sizing

```
RDS / Aurora:
  - Performance Insights 분석
  - read replica 활용 (read 부담 분산)
  - aurora serverless v2 (auto-scale)
  - aurora I/O optimized vs standard

DynamoDB:
  - on-demand vs provisioned
  - auto-scaling
  - reserved capacity

ElastiCache:
  - 적정 node count
  - data tiering (r6gd 옵션)
```

---

## 11. 함정

1. **right-size 만, 트래픽 spike 안 봄** — peak 시 down.
2. **t (burstable) 의 CPU credit** — credit 떨어지면 throttle.
3. **scale-down threshold 만 — too aggressive** — flap.
4. **autoscaler config 없음** — 수동 scale.
5. **HPA + 외부 LB** — 새 pod 인식 안 됨.
6. **Karpenter + critical workload spot** — 갑작스 종료.
7. **dev 끄기 자동화 — secret / DB 도 종료** — startup 문제.

---

## 12. 관련

- [[finops|↑ finops]]
- [[reserved-and-spot]]
- [[kubernetes-cost]]
- [[../kubernetes/autoscaling|↗ k8s autoscale]]
