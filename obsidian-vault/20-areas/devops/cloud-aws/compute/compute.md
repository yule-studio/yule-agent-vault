---
title: "AWS Compute (Hub)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:05:00+09:00
tags:
  - aws
  - compute
  - hub
---

# AWS Compute (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 카테고리 hub |

**[[../cloud-aws|↑ AWS]]**

---

## 1. 서비스

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **EC2** | [[ec2]] | 가상 머신 (VM) — 가장 기본 |
| **ECS** | [[ecs]] | Container — Fargate / EC2 launch |
| **Lambda** | [[lambda]] | Serverless 함수 |

---

## 2. 선택 가이드

| 시나리오 | 추천 |
| --- | --- |
| 전통 응용 / 큰 서버 | EC2 |
| Container + AWS native | ECS (Fargate) |
| Container + 표준 / 이식성 | EKS (별도, Kubernetes hub) |
| Event-driven / 짧은 작업 | Lambda |
| 정적 사이트 | S3 + CloudFront ([[../storage/s3]]) |
| 거대 batch | Batch (별도) |
| HPC / GPU | EC2 GPU instance |

---

## 3. 가격 모델

| 모델 | 의미 |
| --- | --- |
| On-Demand | 사용한 만큼 (기본) |
| Reserved (RI) | 1년/3년 약정 — 30-70% 할인 |
| Savings Plan | 시간당 약정 (더 유연) |
| Spot | 입찰 — 최대 90% 할인 + 강제 종료 |
| Dedicated Host | 단독 물리 서버 (라이선스) |

---

## 4. 관련

- [[../cloud-aws|↑ AWS]]
- [[../network/vpc]] — Compute 가 VPC 안에 있음
- [[../security/iam]] — IAM role 로 권한
