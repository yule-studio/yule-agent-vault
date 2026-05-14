---
title: "product operations hub"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T20:02:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - operations
---

# product operations hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[../product|↑ hub]]**

---

## 1. 영역

| 노트 | 영역 |
| --- | --- |
| [[observability]] | 메트릭 / Prometheus / Grafana |
| [[runbook]] | 5가지 장애 시나리오 |
| [[reconciliation]] | PG 대사 (daily) |

---

## 2. 알람 채널

| 채널 | 용도 |
| --- | --- |
| #payment-alerts | 결제 5xx / fail spike / amount mismatch |
| #fraud | 자동 모더 / 의심 결제 |
| #admin-actions | admin refund / product delete |
| #infra | Kafka lag / DB / Redis |

---

## 3. 관련

- [[../product|↑ hub]]
- [[observability]]
- [[runbook]]
- [[reconciliation]]
