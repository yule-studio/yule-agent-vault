---
title: "Architecture optimization — 비싼 service 회피"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:08:00+09:00
tags: [devops, finops, architecture]
---

# Architecture optimization — 비싼 service 회피

**[[finops|↑ finops]]**

---

## 1. 보이지 않는 비싼 service

```
AWS 신규 사용자 흔한 실수:
  1. NAT Gateway              $32/mo + $0.045/GB processing
  2. Elastic IP unused         $3.6/mo × N
  3. Data transfer cross-AZ    $0.01/GB × 2 (양방향)
  4. Data egress               $0.09/GB (1st GB free)
  5. CloudWatch Logs ingest    $0.50/GB
  6. CloudWatch metric         $0.30/metric/mo
  7. ALB always-on             $16/mo (idle 시도)
  8. Provisioned IOPS          $0.10/IOPS/mo (큰 DB)
  9. Backup retention          $$ (RDS automated)
  10. Reserved capacity unused
```

→ 시니어 = 이들 인지 + 회피 / 최적화.

---

## 2. NAT Gateway 최적화 (★ 큰 절감)

```
문제:
  - $32/mo per AZ × 3 AZ = $96/mo
  - + $0.045 per GB processed
  - traffic 많을수록 폭증

대안:

A. VPC Endpoint (★)
  - S3, DynamoDB → free (gateway endpoint)
  - 기타 AWS service → $0.01/h × VPC + $0.01/GB (interface endpoint)
  - 90% 트래픽 즉시 절감

B. NAT Instance
  - EC2 가 NAT 역할
  - $0.01/h
  - SPOF / 관리 부담
  - 작은 회사 OK

C. fck-nat (★ OSS)
  - https://fck-nat.dev/
  - 더 저렴 (Spot 가능)

D. 같은 region private subnet 만
  - cross-AZ 도 internal CIDR routing
```

```hcl
# S3 Gateway Endpoint (★ 무료)
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${var.region}.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = [aws_route_table.private.id]
}

# DynamoDB Gateway Endpoint (★ 무료)
resource "aws_vpc_endpoint" "dynamodb" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${var.region}.dynamodb"
  vpc_endpoint_type = "Gateway"
}

# ECR / Secrets Manager / SSM (Interface — $$)
resource "aws_vpc_endpoint" "ecr" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.region}.ecr.api"
  vpc_endpoint_type   = "Interface"
  private_dns_enabled = true
  subnet_ids          = aws_subnet.private[*].id
}
```

---

## 3. Data transfer 비용

```
free:
  - same AZ
  - to AWS service in same region (via Gateway Endpoint)
  - CloudFront → user (별도 가격)

cost:
  - cross-AZ same region:  $0.01/GB × 2 (양방향)
  - cross-region:          $0.02/GB (vary)
  - to internet:           $0.09/GB (1st 100GB)
  - CloudFront origin:     $0.02/GB

빅 사례:
  - 1 PB egress = $90,000+
  - cross-AZ replication 1 TB / day = $20/mo × 365 = $7,300/yr
```

전략:
- service 같은 AZ에 (단 AZ 단일 = SPOF)
- region-local 처리
- CloudFront (CDN) 활용
- compression
- 같은 region 의 SaaS (Datadog 등) 사용

---

## 4. serverless vs container vs VM

```
traffic 매우 spotty (분당 0-100 req):
  Lambda (★) — $0 if idle, pay per req
  Fargate Spot — small idle cost
  EC2 — idle cost 큼

traffic 일정 (1000 req/s 항상):
  EC2 RI / Savings Plan — 가장 저렴
  Fargate — 약간 비쌈 but easier
  Lambda — 무지 비쌈 (per-request)

traffic 매우 큼 (10k req/s):
  EC2 RI + Spot mix
  ALB direct routing
```

→ workload 패턴 별 break-even 다름. 시뮬레이션 필요.

---

## 5. 비싼 service vs 저렴 alternative

```
ELB (ALB / NLB):
  $16/mo idle minimum
  → 작은 service 한 ALB 에 multi-host routing

API Gateway:
  $3.50 per million requests + data
  → Lambda function URL (free) or ALB direct

CloudWatch Logs:
  $0.50/GB ingest
  → Loki (자체 host) / Vector + S3

NAT Gateway:
  → VPC Endpoint / fck-nat

Aurora vs RDS:
  Aurora = MySQL/PG 호환 + 더 좋음 but 더 비쌈
  → 작은 / dev = RDS, prod / 큰 = Aurora

Lambda Provisioned Concurrency:
  warm 유지 = idle cost
  → 정말 low latency 필요한 경우만

Step Functions:
  per-transition cost
  → 간단한 flow = SQS + Lambda
```

---

## 6. S3 storage class

```
Standard:             $0.023/GB (default)
Standard-IA:          $0.0125/GB (30일 후, retrieval cost)
Intelligent-Tiering:  자동, no retrieval cost (★ 권장)
Glacier Instant:      $0.004/GB (즉시 access)
Glacier Flexible:     $0.0036/GB (3-5 hour retrieve)
Glacier Deep Archive: $0.00099/GB (12 hour retrieve)
```

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "main" {
  bucket = aws_s3_bucket.main.id
  rule {
    id = "tiering"
    status = "Enabled"

    transition {
      days          = 30
      storage_class = "INTELLIGENT_TIERING"
    }

    transition {
      days          = 90
      storage_class = "GLACIER"
    }

    expiration {
      days = 365   # 1년 후 삭제
    }
  }
}
```

→ **Intelligent-Tiering** = 거의 자동 최적. 작은 file (128KB↓) 만 standard.

---

## 7. CloudWatch Logs → alternative

```
CloudWatch Logs:
  - $0.50/GB ingest
  - $0.03/GB storage
  - $0.005/GB query (Insights)
  - 1 TB / mo = $500+ ingest

Alternative:
  - Loki + S3 (자체 host)
  - Vector + S3 (가벼움)
  - Datadog Logs (다른 $)
  - Splunk

조합:
  - Application log = Loki (검색)
  - System log = CloudWatch (조회 빈도 낮음)
  - 정기 archive = S3 + Athena
```

---

## 8. 정확한 service 선택

```
"a queue 필요"
  - SQS:        per-million-req
  - Kinesis:    shard-hour + data
  - Kafka MSK:  cluster $$ 항상
  → 사용량 별 break-even

"a database"
  - RDS:        instance $$
  - Aurora:     instance + storage scaled
  - DynamoDB:   pay-per-req or capacity
  - Aurora Serverless v2: scale to 0.5 ACU

"object storage"
  - S3:         default
  - EFS:        NFS, 비싸
  - FSx:        Windows / Lustre
```

→ 시니어 = 매번 break-even point 계산.

---

## 9. multi-region strategy

```
benefit:
  - DR
  - latency (가까운 region)

cost:
  - cross-region replication
  - duplicate infra
  - duplicate licenses
  - traffic crossing

전략:
  - 1 primary region (90% traffic)
  - 1 DR region (warm standby — 작게)
  - critical 만 multi-region active-active
```

---

## 10. dev environment 최적화

```
typical waste:
  - 24/7 dev / staging
  - 큰 instance "안전하게"
  - prod 와 동일 size
  - 각 dev 마다 own cluster

best practice:
  - 평일 9-18 + 끄기 (CronJob)
  - 작은 instance (1/4 size)
  - shared dev cluster
  - on-demand cluster (k3d / ephemeral)
  - feature branch → ephemeral env (Vercel 스타일)
```

→ **dev 비용 = prod 의 10% 이하 권장**.

---

## 11. application layer 최적화

```
- compression (gzip / brotli)
- HTTP cache header (CDN cache hit)
- batch API (N+1 X)
- pagination
- lazy load
- connection pooling (RDS proxy)
- read replica
- async / queue 활용
- DB index 적정 (불필요 X)
```

→ application = cloud cost 의 가장 큰 lever.

---

## 12. 함정

1. **NAT Gateway 무관심** — 가장 큰 cost 의외.
2. **multi-AZ 무조건** — 작은 service = single AZ + IaC 자동 복구.
3. **EBS 사용 안 하는 snapshot** — 누적.
4. **EIP unused** — 매번 release.
5. **dev 24/7** — 60%+ 절감 기회 놓침.
6. **log 무조건 CloudWatch** — Loki 절감 큼.
7. **service 큰 거 선택 "안전"** — break-even 계산.

---

## 13. 관련

- [[finops|↑ finops]]
- [[right-sizing]]
- [[data-transfer-cost]]
- [[../cloud-aws/cloud-aws|↗ AWS]]
