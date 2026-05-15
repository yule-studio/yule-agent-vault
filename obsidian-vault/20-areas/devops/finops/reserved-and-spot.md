---
title: "Reserved / Savings Plan / Spot — 약속 기반 할인"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:06:00+09:00
tags: [devops, finops, reserved, spot]
---

# Reserved / Savings Plan / Spot — 약속 기반 할인

**[[finops|↑ finops]]**

---

## 1. 모델

```
on-demand:    가장 비쌈, flexibility 최고
Reserved:     1년 / 3년 약속 → 30-72% 할인
Savings Plan: 비슷 + 더 flexible
Spot:         최대 90% 할인 — 갑자기 종료될 수도
```

→ 시니어 = 이들의 mix 디자인.

---

## 2. Reserved Instance (RI)

```
약속:
  - instance type (m5.large)
  - region
  - tenancy (default)
  - 1년 또는 3년

지불:
  - All upfront      72% 할인
  - Partial upfront  64% 할인
  - No upfront       55% 할인

종류:
  - Standard RI      가장 큰 할인, 거의 modify 불가
  - Convertible RI   modify 가능, 작은 할인 (54-66%)
```

→ **단점: instance type / family lock-in**.

---

## 3. Savings Plan (★ 권장)

```
Compute Savings Plan:
  - 시간당 $X 약속 (1년 / 3년)
  - EC2 / Fargate / Lambda 모두 적용
  - region / family 자유
  - 66% 할인까지

EC2 Instance Savings Plan:
  - 특정 family (예: m5) 약속
  - 72% 할인까지
  - 변경 어려움

SageMaker Savings Plan:
  - ML
```

→ **Compute Savings Plan = 가장 flexible**. 거의 모든 회사에 적합.

---

## 4. coverage 전략

```
일정한 base load 식별:
  - 24/7 trafiic 의 70% = stable
  - peak 의 추가 30% = on-demand

전략:
  base 70% → 3yr All upfront Savings Plan (72% 할인)
  peak 30% → on-demand
  batch / non-critical → Spot

목표:
  reservation coverage > 80%
  utilization > 95%
```

---

## 5. coverage 측정

```bash
# AWS Cost Explorer
# Reservation Coverage report
# Savings Plan Coverage report

# CLI
aws ce get-reservation-coverage \
    --time-period Start=2026-05-01,End=2026-05-31 \
    --granularity MONTHLY

aws ce get-savings-plans-coverage \
    --time-period Start=2026-05-01,End=2026-05-31
```

→ < 80% = 더 사야. > 100% (unused) = 줄여야.

---

## 6. RI 권장 (AWS Compute Optimizer)

```
14일 사용 분석:
  "이 m5.large 12개 모두 3yr All upfront 권장 → $X 절감"
  
검토 → 구매.
```

→ ML 기반 권장. 잘못된 사면 lock-in.

---

## 7. Spot 활용 (★ 시니어 패턴)

```
on-demand: $0.10/h
Spot:      $0.02/h (80% 할인)

단점:
  - 2분 notice 후 종료
  - capacity 부족 시 발급 안 됨

따라서:
  ✓ batch / video encoding / ML training
  ✓ web (stateless) + 다양한 instance type mix
  ✓ k8s with PDB
  ✗ DB master
  ✗ critical singleton
```

---

## 8. Spot best practice

```
1. 다양한 instance type
   m5.large + m5a.large + m4.large + m5n.large + ...
   → capacity 한 type 부족 → 다른 자동

2. AZ 다양화
   us-east-1a + 1b + 1c

3. 짧은 task
   2분 안에 graceful shutdown 가능

4. checkpoint
   batch task 는 진행 checkpoint 저장

5. Spot Fleet / Karpenter
   여러 type / AZ 자동 mix

6. ASG 의 mixed instance policy
   on-demand 30% + spot 70%
```

---

## 9. k8s + Spot

```yaml
# Karpenter NodePool
spec:
  template:
    spec:
      requirements:
        - {key: karpenter.sh/capacity-type, operator: In, values: [spot]}
        - {key: node.kubernetes.io/instance-type, operator: In, values: [m5.large, m5a.large, m4.large, c5.large]}
      taints:
        - key: spot
          effect: NoSchedule
```

```yaml
# Deployment 의 toleration
spec:
  template:
    spec:
      tolerations:
        - key: spot
          operator: Exists
          effect: NoSchedule
      # PDB 로 갑자기 종료 보호
```

```yaml
# PodDisruptionBudget
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: {name: web}
spec:
  minAvailable: 80%
  selector: {matchLabels: {app: web}}
```

---

## 10. spot termination handler

```yaml
# aws-node-termination-handler
helm install aws-node-termination-handler \
    eks/aws-node-termination-handler \
    --set enableSpotInterruptionDraining=true \
    --set enableRebalanceMonitoring=true
```

→ 2분 notice 받으면 node drain 시작 → graceful.

---

## 11. database + RI

```
RDS RI:
  - DB instance class 약속
  - up to 60% 할인
  - flexibility (size 변경 OK in family)

Aurora RI:
  - Aurora-specific
  - 일부 modify
  
DynamoDB Reserved:
  - read/write capacity 약속
  - 53-77% 할인
```

→ DB 는 거의 늘 RI (안정 traffic).

---

## 12. Savings Plan 구매 가이드

```
1. 분석: 최근 3개월 평균 사용량
   "EC2 평균 $5,000 / mo"

2. base load 식별:
   "최저 $3,500 / mo 가 항상"

3. 구매:
   $3,500 / 730 hours = $4.79 / hour
   1yr Partial upfront: 약 30% 할인
   
   = 월 $2,450 (saving $1,050 / mo = $12,600 / yr)

4. 다음 분기 검토 → 추가
```

→ "100% cover" 시도하지 말 것. 90% 가 sweet spot.

---

## 13. 함정

1. **3yr All upfront** — workload 변경 시 lock-in.
2. **Standard RI** — convertible 보다 큰 할인이지만 modify 불가.
3. **Spot critical workload** — 종료 시 영향.
4. **Spot 의 single instance type** — capacity 부족 시 발급 X.
5. **Coverage 100% 목표** — 약간 over-buy → unused.
6. **RI / SP 만료 잊음** — 비용 폭증.
7. **k8s + Spot + PDB 없음** — node 다 termination 시 모두 down.

---

## 14. 관련

- [[finops|↑ finops]]
- [[right-sizing]]
- [[kubernetes-cost]]
- [[../kubernetes/autoscaling|↗ k8s autoscale]]
