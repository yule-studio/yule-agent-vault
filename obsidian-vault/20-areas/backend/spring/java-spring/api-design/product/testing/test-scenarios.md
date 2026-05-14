---
title: "Test scenarios — AC → test case 매핑"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:52:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - testing
---

# Test scenarios — AC → test case 매핑

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[testing|↑ hub]]**

> 52 AC ([[../requirements]]) 의 모든 항목 → 통합 테스트 1 case.

---

## 1. AC → 테스트 매핑 (요약)

| AC | 테스트 | 도구 |
| --- | --- | --- |
| P-1 admin 등록 | ProductCrudIT.adminCanCreate | TC PG |
| P-2 상태 전이 | ProductStatusTransitionTest | Unit |
| P-5 XSS | XssSanitizationTest | Unit |
| K-1 회원만 | CheckoutIT.guestRejected | TC |
| O-1 atomic 차감 | InventoryConcurrencyIT.100threads | TC Redis |
| O-3 100 요청 / 1 재고 | InventoryConcurrencyIT.onlyOneSucceeds | TC |
| O-4 timeout 복원 | OrderTimeoutWorkerIT | TC |
| PAY-1 confirm 흐름 | PaymentConfirmIT.tossSandbox | WireMock |
| PAY-2 idempotency | PaymentConfirmIT.duplicateKeyReturnsSame | TC Redis |
| PAY-3 amount 변조 | PaymentConfirmIT.amountTamperedRejected | Unit |
| PAY-4 webhook 서명 | TossWebhookIT.invalidSignature401 | Unit |
| PAY-5 webhook dedup | TossWebhookIT.duplicateEventOnce | TC |
| PAY-7 PG timeout | PaymentConfirmIT.pgTimeoutFails | WireMock |
| R-1 전체 환불 | RefundIT.fullCancel | TC |
| R-2 부분 환불 | RefundIT.partialThenFull | TC |
| R-3 환불 멱등 | RefundIT.duplicateIdempotent | TC |
| R-4 디지털 다운 후 X | RefundIT.digitalDownloadedRejected | TC |
| D-1 워터마크 enqueue | DigitalDeliveryListenerIT | TC |
| D-2 워터마크 worker | WatermarkWorkerIT.pdfGenerated | TC |
| D-5 토큰 만료 | DownloadIT.expiredToken401 | TC |
| D-6 audit | DownloadIT.recordsAccess | TC |
| D-7 환불 revoke | RefundIT.digitalRevokesAccess | TC |
| S-3 대사 | SettlementReconciliationIT | TC |

---

## 2. F10+ Kafka 테스트

| 시나리오 | 테스트 |
| --- | --- |
| outbox 만 INSERT (트랜잭션 rollback 시) | OutboxIT.rollbackNoEvent |
| outbox worker → Kafka send | OutboxIT.workerPublishes |
| consumer dedup | ConsumerIT.duplicateMessageOnce |
| DLQ → 5회 retry 후 | DltIT.movesToDlq |
| partition 순서 | PartitionOrderIT |

→ EmbeddedKafka.

---

## 3. 관련

- [[testing|↑ hub]]
- [[../requirements]]
- [[unit-tests]]
- [[integration-tests]]
- [[pg-mock-tests]]
