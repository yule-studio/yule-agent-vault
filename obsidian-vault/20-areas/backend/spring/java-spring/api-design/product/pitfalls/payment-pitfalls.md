---
title: "Payment 함정 — PG / webhook / 금액 변조"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T20:16:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - pitfalls
  - payment
---

# Payment 함정 — PG / webhook / 금액 변조

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[pitfalls|↑ hub]]**

---

## 함정 1 — amount 변조 (FE → server)
1억 결제를 100원으로.
→ 서버에서 order.totalAmount 와 비교.

## 함정 2 — Idempotency-Key 없음
중복 결제.
→ 헤더 + Redis SETNX + DB UNIQUE.

## 함정 3 — webhook 서명 검증 X
fake event 처리.
→ HMAC-SHA-256.

## 함정 4 — webhook idempotency X
같은 event 2번 → 결제 2번 approve.
→ event_id UNIQUE.

## 함정 5 — PG 호출 timeout 길게 (30s)
사용자 화면 stuck + DB 락.
→ 5s + webhook 보강.

## 함정 6 — paymentKey 검증 없이 신뢰
attacker fake key.
→ PG /payments/{key} 재조회.

## 함정 7 — 카드 정보 서버 저장
PCI-DSS 위반.
→ PG SDK 만 처리.

## 함정 8 — 단일 PG 의존 (이중화 X)
PG 장애 시 매출 0.
→ F8+ KCP fallback.

## 함정 9 — pg_response raw 안 저장
분쟁 evidence 없음.
→ JSONB raw.

## 함정 10 — 정산 mismatch 모니터링 X
회계 사고 늦게 발견.
→ daily 대사 + alert.

## 함정 11 — successUrl 변조
사용자가 결제 안 했는데 success.
→ paymentKey 서버 재검증.

## 함정 12 — PG IP 화이트리스트만
PG IP 변경 시 webhook 못 받음.
→ HMAC + IP 이중.

## 함정 13 — webhook 5xx 응답
PG 무한 retry → DDoS.
→ 처리 실패도 200 + dlq.

## 함정 14 — PG TZ 혼동
정산 mismatch.
→ KST 명시.

## 함정 15 — F10+ Kafka outbox 없이 직접 send
DB commit 후 Kafka timeout — 이벤트 손실.
→ outbox.

---

## 관련

- [[pitfalls|↑ hub]]
- [[../design-decisions/payment-flow]]
- [[../security/idempotency-key]]
- [[../security/webhook-signature]]
- [[../security/pci-dss]]
- [[../design-decisions/kafka-event-driven]]
