---
title: "Cost visibility — tag / dashboard / billing"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:02:00+09:00
tags: [devops, finops, cost, tag]
---

# Cost visibility — tag / dashboard / billing

**[[finops|↑ finops]]**

---

## 1. tag 가 모든 것의 시작

```
"누가 / 어디서 / 무엇 / 왜"
  ↑ tag 가 답.

tag 없이 시도:
  - 비용 분배 불가
  - "누구 거?" 모름 → 끌 수 없음
  - chargeback 불가
```

→ **tag 정책 = FinOps 의 토대**.

---

## 2. 표준 tag 정책 (★)

```
Environment   : prod / staging / dev
Team          : platform / backend / frontend / data
Service       : user-api / order-api / billing
Owner         : alice@example.com
Project       : Q3-launch
CostCenter    : engineering / marketing
ManagedBy     : terraform / manual / k8s
CreatedAt     : 2026-05-15
ExpiresAt     : 2026-08-15  (자동 cleanup hint)
```

→ 회사 표준 7-10 tag. 모두 필수.

---

## 3. tag 강제 (★)

### AWS: tag policy + IAM

```json
{
  "Effect": "Deny",
  "Action": ["ec2:RunInstances"],
  "Resource": "*",
  "Condition": {
    "Null": {
      "aws:RequestTag/Environment": "true",
      "aws:RequestTag/Team": "true"
    }
  }
}
```

→ tag 없으면 자원 생성 거부.

### Terraform default tag

```hcl
provider "aws" {
  default_tags {
    tags = {
      ManagedBy   = "terraform"
      Environment = var.env
      Team        = var.team
    }
  }
}
```

→ 모든 자원 자동 tag.

### policy (OPA / Sentinel)

```rego
deny[msg] {
  resource := input.resource.aws_instance[_]
  not resource.tags.Environment
  msg := "Environment tag required"
}
```

---

## 4. AWS Cost Explorer / Billing

```bash
# CLI
aws ce get-cost-and-usage \
    --time-period Start=2026-05-01,End=2026-05-31 \
    --granularity DAILY \
    --metrics UnblendedCost \
    --group-by Type=TAG,Key=Team

# Console
Cost Explorer → group by tag:Team / Service
```

→ 매월 첫째 주 = 비용 review meeting.

---

## 5. Cost Allocation Tag

```
AWS Console → Billing → Cost allocation tags
  → tag activate (24h 후 reporting 에 등장)
```

→ active 안 하면 Cost Explorer 에서 group X.

---

## 6. budget / alert (★)

```bash
# AWS Budget
aws budgets create-budget \
    --account-id 123 \
    --budget '{
      "BudgetName": "prod-monthly",
      "BudgetLimit": {"Amount": "10000", "Unit": "USD"},
      "TimeUnit": "MONTHLY",
      "BudgetType": "COST",
      "CostFilters": {"TagKeyValue": ["user:Environment$prod"]}
    }' \
    --notifications-with-subscribers '[
      {"Notification": {"NotificationType": "ACTUAL", "ComparisonOperator": "GREATER_THAN", "Threshold": 80},
       "Subscribers": [{"SubscriptionType": "EMAIL", "Address": "ops@example.com"}]
      },
      {"Notification": {"NotificationType": "FORECASTED", "ComparisonOperator": "GREATER_THAN", "Threshold": 100},
       "Subscribers": [{"SubscriptionType": "EMAIL", "Address": "cfo@example.com"}]
      }
    ]'
```

→ 50% / 80% / 100% / 120% 단계 alert.

---

## 7. Cost Anomaly Detection (★)

```
AWS Cost Anomaly Detection (free):
  - ML 이 평소 패턴 학습
  - 이상 비용 spike 자동 detect
  - email / Slack alert

예:
  "지난 24시간 S3 비용이 평소의 5배" → alert
```

→ 매월 끝나서 알기 vs 즉시 알기. 큰 차이.

---

## 8. CUR (Cost and Usage Report)

```
가장 raw / 자세한 billing data → S3 hourly
  - parquet / csv
  - all field (instance-hour 단위)

분석:
  - Athena 로 SQL
  - QuickSight / Tableau / Grafana dashboard
  - 자체 BI 통합
```

```sql
SELECT
    line_item_product_code,
    line_item_resource_id,
    resource_tags_user_team,
    SUM(line_item_unblended_cost) as cost
FROM cur_table
WHERE line_item_usage_start_date >= DATE '2026-05-01'
GROUP BY 1, 2, 3
ORDER BY cost DESC
LIMIT 50;
```

→ 시니어 = SQL 으로 자체 query.

---

## 9. dashboard

```
Grafana / Datadog / 자체:
  Row 1: 전체 spend 추세 (12개월)
  Row 2: 서비스 별 (EC2 / RDS / S3 ...)
  Row 3: 팀 별 (tag:Team)
  Row 4: 환경 별 (prod / staging / dev)
  Row 5: top-10 resource (큰 비용)
  Row 6: cost per transaction (business 지표)
  Row 7: budget vs actual
  Row 8: 예측 (이번 달 끝)
  Row 9: savings opportunity (RI 사용률)
```

---

## 10. multi-cloud 통합

| | 무엇 |
| --- | --- |
| **Cloudability** | enterprise SaaS |
| **CloudHealth** | VMware (multi-cloud) |
| **Vantage** | UI 친화 |
| **Spot.io** | optimization 자동 |
| **Kubecost / OpenCost** | k8s 전용 |
| **AWS Cost Explorer** | AWS only |
| **자체** | CUR + 자체 BI |

→ AWS only = native, multi-cloud = Cloudability / CloudHealth.

---

## 11. weekly / monthly review

```
Weekly (FinOps lead):
  - 새 자원 review
  - anomaly check
  - budget burn rate
  - 사용 안 하는 자원 list

Monthly (전체 팀):
  - 팀별 비용 review
  - 큰 자원 review
  - savings opportunity
  - 다음달 forecast
  - action item
```

→ "끝나서 청구서 보고 놀람" 방지.

---

## 12. 함정

1. **tag 강제 안 함** — 누가 만든지 모름.
2. **default tag 만 의존** — manual / console 자원은 빠짐.
3. **Cost Allocation tag activate 안 함** — group 불가.
4. **anomaly 무시** — 매주 review.
5. **billing alert 너무 늦음** — 80% 이후엔 늦음.
6. **forecast 못 봄** — 다음 달 예측 X.
7. **CUR 만 보고 action 안 함** — 정기 review.

---

## 13. 관련

- [[finops|↑ finops]]
- [[right-sizing]]
- [[reserved-and-spot]]
- [[kubernetes-cost]]
