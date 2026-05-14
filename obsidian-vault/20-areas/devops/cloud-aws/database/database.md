---
title: "AWS Database (Hub)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:07:00+09:00
tags:
  - aws
  - database
  - hub
---

# AWS Database (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 카테고리 hub |

**[[../cloud-aws|↑ AWS]]**

---

## 1. 서비스

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **RDS** | [[rds]] | 매니지드 RDB (Postgres / MySQL / MariaDB / Oracle / SQL Server) |
| **Aurora** | [[aurora]] | AWS 자체 분산 RDB — Postgres / MySQL 호환 |
| **DynamoDB** | [[dynamodb]] | 매니지드 KV / Document NoSQL |
| **ElastiCache** | (별도) | 매니지드 Redis / Memcached |

---

## 2. 선택

| 시나리오 | 추천 |
| --- | --- |
| 일반 OLTP / 이식성 | RDS Postgres / MySQL |
| 거대 워크로드 / 분산 storage | Aurora |
| Key-Value / 무한 확장 / AWS native | DynamoDB |
| 캐시 / 세션 / 분산 락 | ElastiCache (Redis) |
| 분석 / DW | Redshift (별도) |
| 시계열 | Timestream (별도) |
| 그래프 | Neptune (별도) |

---

## 3. DB 자체 운영 노하우는

[[../../../database/database|↗ 20-areas/database]] — Postgres / MySQL / Redis 자체.
여기는 **AWS 측면** (관리형 옵션 / 결제 / multi-AZ / backup 등).

---

## 4. 관련

- [[../cloud-aws|↑ AWS]]
- [[../../../database/database]]
