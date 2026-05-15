---
title: "FinOps — Cloud 비용 관리 ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:00:00+09:00
tags: [area, devops, finops, cost]
---

# FinOps — Cloud 비용 관리 ★

**[[../devops|↑ devops]]**

---

## 1. 왜

```
Cloud 비용 매년 30-50% 증가:
  - 잘못된 instance type
  - 사용 안 하는 자원
  - 비싼 service (NAT GW, traffic egress)
  - over-provisioned database
  - dev 환경 24/7
  - data egress
  - 미사용 LB / NAT

→ 시니어 = 코드 + 운영 + 비용 책임.
```

→ FinOps Foundation — "**FinOps** = 비용을 책임 있게 다루는 cultural practice".

---

## 2. FinOps 의 3 phase

```
1. Inform (보이게)
   - 누가 / 무엇 / 얼마
   - tag / dashboard / report

2. Optimize (줄이기)
   - 사용 안 하는 것 끄기
   - right-sizing
   - reserved / spot
   - architecture 개선

3. Operate (정착)
   - 자동화
   - 정기 review
   - 책임 분담
   - KPI 추적
```

---

## 3. 하위 영역

- [[cost-visibility]] — tagging / billing / dashboard
- [[right-sizing]] — 적정 size / autoscaling
- [[reserved-and-spot]] — Reserved / Savings Plan / Spot
- [[architecture-optimization]] — service 선택 / serverless / 비싼 service 제거
- [[kubernetes-cost]] — k8s 의 cost 특수 (Karpenter / Spot / KEDA)
- [[data-transfer-cost]] — egress / inter-AZ / VPC peering
- [[storage-cost]] — S3 lifecycle / EBS / 옛 snapshot
- [[showback-chargeback]] — 팀별 비용 배분
- [[finops-tools]] — Cloud native / Cloudability / Vantage 등
- [[pitfalls]]

---

## 4. 핵심 원칙

1. **모두의 책임** — engineer + finance + product.
2. **timely / accurate** — 실시간 / 정확한 데이터.
3. **business value 중심** — 비용 vs 가치.
4. **cloud variable cost 활용** — on-demand / RI / Spot.
5. **자동화** — 사람 review 만 X.
6. **decentralize** — 팀별 ownership.

---

## 5. 흔한 cost 분포 (AWS 표준)

```
EC2 / EKS:           40-60%
RDS / Aurora:        15-25%
S3:                  5-15%
Data transfer:       10-20%  ← 의외로 큼
ElastiCache / DynamoDB: 5-10%
Lambda / API GW:     2-10%
Networking (NAT/LB): 5-15%   ← 보이지 않는 비용
기타:                 5-10%
```

→ 비용 큰 영역부터 attack.

---

## 6. 비용 KPI (시니어 측정)

| | 무엇 |
| --- | --- |
| **Total Cloud Spend** | 절대 비용 |
| **Cost per Customer** | 사용자당 |
| **Cost per Transaction** | 트랜잭션당 (가장 의미) |
| **Cost / Revenue Ratio** | 매출 대비 |
| **Cost variance** | 예측 vs 실제 |
| **Reservation coverage** | RI / Savings Plan 의 사용률 |
| **Spot %** | Spot 사용 비율 |
| **Idle resource %** | 사용 안 하는 % |
| **Tagged %** | tag 적용된 % |

→ "$1M 비용" 보다 "trx 당 $0.001" 가 의미.

---

## 7. 학습 순서

1. Day 1: [[cost-visibility]] (tag + Cost Explorer)
2. Day 2: [[right-sizing]] (CloudWatch / Datadog 분석)
3. Day 3: [[reserved-and-spot]] (Savings Plan 도입)
4. Day 4: [[kubernetes-cost]] (Karpenter / OpenCost)
5. Day 5: [[data-transfer-cost]] + [[architecture-optimization]]

---

## 8. 시니어 의사결정 단골

```
"NAT Gateway $33/mo + traffic 비싸"
  → VPC endpoint 사용 (S3 / ECR 등)
  → 옵션: NAT Instance (관리 부담)

"egress 비쌈 (cross-region)"
  → CloudFront / 동일 region 의존성 / data 캐시

"prod RDS overprovisioned"
  → Performance Insights 분석 → 1 size down

"dev 환경 24/7"
  → 평일 9-18 만 (Lambda 자동 stop/start)
  → 주말 끄기

"EKS node 50% idle"
  → Karpenter + Spot

"S3 1년+ 옛 data"
  → S3 Intelligent-Tiering / Glacier

"unused Elastic IP"
  → 5 unused = $36/mo
```

---

## 9. 관련

- [[../devops|↑ devops]]
- [[../cloud-aws/cloud-aws|↗ AWS]]
- [[../kubernetes/autoscaling|↗ k8s autoscale]]
- [[../platform-engineering/developer-experience|↗ DX cost]]
