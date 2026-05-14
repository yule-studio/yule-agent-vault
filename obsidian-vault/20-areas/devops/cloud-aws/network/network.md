---
title: "AWS Network (Hub)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:08:00+09:00
tags:
  - aws
  - network
  - hub
---

# AWS Network (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 카테고리 hub |

**[[../cloud-aws|↑ AWS]]**

---

## 1. 서비스

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **VPC** | [[vpc]] | 가상 네트워크 — 모든 자원의 base |
| **ALB / NLB** | [[alb]] | Load Balancer (L7 / L4) |
| **CloudFront** | [[cloudfront]] | CDN — 전 세계 edge cache |
| **Route 53** | [[route53]] | DNS + health check + traffic policy |

---

## 2. 토폴로지 단순 예

```
인터넷
  ↓
Route 53 (DNS)
  ↓
CloudFront (CDN) — 정적
  ↓
ALB (load balancer)
  ↓
VPC public subnet — NAT GW
       │
       └─ private subnet — EC2 / ECS / RDS
```

---

## 3. 비용 함정

- **NAT Gateway** — 매우 비쌈 ($32/월 + GB-out)
- **Data Transfer Out** — 인터넷으로 나가는 트래픽
- **VPC Endpoint** — S3 / DynamoDB 트래픽이 NAT 안 거치게
- **PrivateLink** — 다른 VPC 의 서비스에 private 접근

---

## 4. 관련

- [[../cloud-aws|↑ AWS]]
- [[../security/iam]] — Security Group 도 일부
- [[../../../computer-science/network/network|↗ network 이론]]
