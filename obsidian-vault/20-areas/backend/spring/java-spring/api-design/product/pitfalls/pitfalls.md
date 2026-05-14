---
title: "product pitfalls hub"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T20:12:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - pitfalls
---

# product pitfalls hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[../product|↑ hub]]**

> 영역 별 함정 카탈로그. 각 함정 = 사고 사례 + 방어.

---

## 1. 영역

| 노트 | 영역 |
| --- | --- |
| [[concurrency-pitfalls]] | 재고 / 이중 결제 / 락 |
| [[payment-pitfalls]] | PG / webhook / 금액 변조 |
| [[digital-delivery-pitfalls]] | 워터마크 / 토큰 / 도용 |
| [[refund-pitfalls]] | 환불 / 정산 / 디지털 revoke |

---

## 2. 관련

- [[../product|↑ hub]]
- [[../transactions]] — race 분석
