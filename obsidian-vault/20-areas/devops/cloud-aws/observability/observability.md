---
title: "AWS Observability (Hub)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:11:00+09:00
tags:
  - aws
  - observability
  - monitoring
  - hub
---

# AWS Observability (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 카테고리 hub |

**[[../cloud-aws|↑ AWS]]**

---

## 1. 서비스

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **CloudWatch** | [[cloudwatch]] | Metric / Log / Alarm / Dashboard |
| **CloudTrail** | [[cloudtrail]] | 모든 AWS API 호출 audit log |

---

## 2. 3 축

```
Metric    — 수치 시계열 (CPU/메모리/요청 수)
Log       — 텍스트 이벤트
Trace     — 분산 추적 (서비스 A → B → C)
```

| AWS | 외부 대안 |
| --- | --- |
| CloudWatch Metrics | Prometheus + Grafana |
| CloudWatch Logs | ELK / Loki |
| X-Ray | Jaeger / Tempo |

---

## 3. 추가 서비스

- **X-Ray** — 분산 trace (마이크로서비스)
- **CloudWatch Logs Insights** — log query
- **Container Insights** — ECS / EKS metric
- **OpenTelemetry on AWS** — vendor-neutral

---

## 4. 관련

- [[../cloud-aws|↑ AWS]]
- [[../monitoring/monitoring]]
- [[../sre/sre]]
