---
title: "Concurrency 함정 — 재고 / 이중 결제 / 락"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T20:14:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - pitfalls
  - concurrency
---

# Concurrency 함정 — 재고 / 이중 결제 / 락

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[pitfalls|↑ hub]]**

---

## 함정 1 — DB UPDATE WHERE stock >= ? 만 (락 X)
동시 100 요청 / 재고 1 — race.
→ Redis Lua atomic 또는 SELECT FOR UPDATE.

## 함정 2 — Redis 만 (DB X)
Redis 장애 시 재고 손실.
→ DB 동기 cron 15분.

## 함정 3 — 결제 후 재고 차감 (주문 후 X)
결제 완료 후 재고 0 → overselling.
→ 주문 시점 차감.

## 함정 4 — 결제 실패 / timeout 복원 X
재고 영구 잠김.
→ scanExpired worker.

## 함정 5 — 같은 user 동시 결제
사용자 더블 클릭 → 결제 2번.
→ Idempotency-Key + UNIQUE (order_id) in payments.

## 함정 6 — Optimistic version 없음
lost update.
→ payment / order / inventory 모두 version.

## 함정 7 — 옵션 (SKU) 단위 분리 X
red XL SOLD_OUT but blue M 도 표시.
→ SKU 별 inventory.

## 함정 8 — Worker 다중 실행 (ShedLock 없음)
같은 row 2번 처리.
→ ShedLock + SKIP LOCKED.

## 함정 9 — DB FOR UPDATE 의 deadlock
순서 다른 lock → deadlock.
→ id 순서 고정 또는 SKIP LOCKED.

---

## 관련

- [[pitfalls|↑ hub]]
- [[../transactions]]
- [[../design-decisions/inventory-strategy]]
- [[../../distributed-lock|↗ distributed-lock]]
