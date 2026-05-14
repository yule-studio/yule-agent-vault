---
title: "AWS Storage (Hub)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:06:00+09:00
tags:
  - aws
  - storage
  - hub
---

# AWS Storage (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 카테고리 hub |

**[[../cloud-aws|↑ AWS]]**

---

## 1. 서비스

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **S3** | [[s3]] | Object storage — 거의 모든 데이터의 home |
| **EBS** | [[ebs]] | Block storage — EC2 의 디스크 |

---

## 2. 종류 (Object / Block / File)

| 종류 | AWS | 의미 |
| --- | --- | --- |
| **Object** | S3 | URL 로 접근, 무한 확장, latency ↑ |
| **Block** | EBS | OS 가 디스크처럼, IOPS 빠름 |
| **File** | EFS / FSx | NFS / SMB share |

---

## 3. 비용 (한국 region, 대략)

| 종류 | $/GB·월 |
| --- | --- |
| S3 Standard | $0.023 |
| S3 Intelligent-Tiering | 동적 |
| S3 Glacier Deep Archive | $0.00099 |
| EBS gp3 | $0.08 |
| EBS io2 | $0.125 + IOPS 추가 |
| EFS Standard | $0.30 |

---

## 4. 관련

- [[../cloud-aws|↑ AWS]]
- [[../database/database]] — DB 의 스토리지
