---
title: "Refund 함정 — 환불 / 정산 / revoke"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T20:20:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - pitfalls
  - refund
---

# Refund 함정 — 환불 / 정산 / revoke

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[pitfalls|↑ hub]]**

---

## 함정 1 — 멱등 X
중복 환불 = 손실.
→ Idempotency-Key + refunds UNIQUE.

## 함정 2 — 디지털 다운 후 환불 허용
부당 이득.
→ digital_deliveries.downloaded_at 검증.

## 함정 3 — 환불 후 다운로드 가능
revoke 누락.
→ AFTER_COMMIT permission revoke + token DELETE.

## 함정 4 — payment.amount 차감 (immutable X)
회계 mismatch.
→ refunds 합산으로 잔여 계산.

## 함정 5 — 환불 합계 > amount
부정합.
→ application 검증.

## 함정 6 — reason 없음 (audit)
누가 왜 환불했는지 X.
→ NOT NULL + audit row.

## 함정 7 — 적립금 환불 시 처리 X
사용자 부당 이득 / 손실.
→ 정책 명확 + 코드 적용.

## 함정 8 — 쿠폰 복원 정책 불명확
사용자 무한 재사용.
→ 정책 (기본 X).

## 함정 9 — 정산 adjustment 누락
다음 달 mismatch.
→ settlement_adjustments INSERT.

## 함정 10 — PG cancel 후 DB rollback
state mismatch.
→ PG 후 DB / 실패 시 alert + 수동 대사.

## 함정 11 — 환불 timeout 처리 X
PG cancel 도중 타임아웃 — 응답 모름.
→ PG /payments/{key} 재조회.

## 함정 12 — 부분 환불 후 디지털 access 그대로
일부만 환불받고 책 가져감.
→ 정책: 부분 환불 시 access 유지 vs revoke 결정.

---

## 관련

- [[pitfalls|↑ hub]]
- [[../design-decisions/refund-policy]]
- [[../implementation/refund-impl]]
- [[../database/refunds-table]]
