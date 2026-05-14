---
title: "product transactions — race / 락 분석"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:08:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - transactions
  - concurrency
---

# product transactions — race / 락 분석

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[product|↑ hub]]**

> "어떤 시나리오에서 race / lost update / overselling 일어나나" 분석 + 방어.

---

## 1. 격리 수준

| 영역 | 격리 | 이유 |
| --- | --- | --- |
| 상품 CRUD | READ_COMMITTED | 일반 |
| 주문 생성 | READ_COMMITTED + Redis Lua | 재고 atomic |
| 결제 confirm | READ_COMMITTED + Idempotency | PG 외부 의존 |
| 환불 | READ_COMMITTED + UNIQUE refund(idempotency) | 중복 방어 |
| 디지털 워터마크 worker | READ_COMMITTED + ShedLock | 단일 worker |
| 통계 / 정산 | REPEATABLE_READ | 일관 snapshot |

---

## 2. race 시나리오 (각 단계)

### 2.1 재고 차감 race

| 시나리오 | 발생 | 방어 |
| --- | --- | --- |
| 동시 100 주문 / 재고 1 | 99 명이 결제 후 재고 0 | Redis Lua atomic |
| 결제 fail 시 복원 + 다른 주문 차감 | 음수 재고 | INCRBY (음수 방지) |
| Redis 장애 후 DB fallback | inconsistency | 매 15분 cron 동기 |
| 옵션 (SKU) 별 race | 같은 SKU 동시 차감 | per-SKU Redis key |

자세히: [[design-decisions/inventory-strategy]] · [[pitfalls/concurrency-pitfalls]].

### 2.2 결제 race

| 시나리오 | 발생 | 방어 |
| --- | --- | --- |
| 사용자 더블 클릭 → /confirm 2회 | 결제 2배 | Idempotency-Key |
| FE 의 amount 변조 | 1억 → 100원 | 서버 재계산 + 비교 |
| /confirm 도중 webhook 먼저 도착 | 상태 race | optimistic version + 멱등 |
| 같은 paymentKey 다른 order | 결제 도용 | UNIQUE (pg_provider, pg_payment_key) |

자세히: [[design-decisions/payment-flow]] · [[pitfalls/payment-pitfalls]].

### 2.3 환불 race

| 시나리오 | 발생 | 방어 |
| --- | --- | --- |
| CS 의 환불 클릭 2회 | 2배 환불 = 손실 | Idempotency + refunds UNIQUE |
| 환불 후 다운로드 | 부당 이득 | AFTER_COMMIT revoke |
| 부분 환불 합계 > payment.amount | 부정합 | application 검증 |
| 환불 + 디지털 다운 동시 | 다운로드 카운트 race | row lock (FOR UPDATE) |

### 2.4 디지털 worker race

| 시나리오 | 발생 | 방어 |
| --- | --- | --- |
| Worker 다중 실행 | 같은 row 2회 워터마크 | ShedLock |
| 다운로드 + revoke 동시 | revoke 실패 | revoked_at 검증 |
| GDrive API rate limit | worker 폭주 | exp backoff + max 5회 |

### 2.5 쿠폰 race

| 시나리오 | 발생 | 방어 |
| --- | --- | --- |
| 같은 쿠폰 동시 사용 | usage_count 초과 | UPDATE WHERE usage < limit + version |
| 1명 N번 사용 | per_user_limit 위반 | UNIQUE (user_id, coupon_id, order_id) |

---

## 3. 트랜잭션 경계 (4가지 흐름)

### 3.1 주문 생성

```
@Transactional (REQUIRED)
1. order INSERT
2. order_items INSERT
3. Redis 재고 차감 (외부 — but atomic)
4. coupon redemption INSERT
5. publishEvent (AFTER_COMMIT)
```

→ Redis 차감 실패 시 throw → DB rollback.

### 3.2 결제 confirm

```
@Transactional (REQUIRED)
1. Idempotency check
2. amount 검증
3. PG 호출 (외부)              ← timeout 위험
4. payment.approve() + order.confirmPayment()
5. payment_transactions INSERT
6. publishEvent
```

→ PG 호출 실패 시 throw → DB rollback. PG 가 승인했는데 DB rollback = state mismatch → webhook 으로 보강.

### 3.3 환불

```
@Transactional (REQUIRED)
1. payment.cancel()
2. refund INSERT
3. order.refund()
4. settlement_adjustments INSERT
5. AFTER_COMMIT: PG cancel + GDrive revoke
```

### 3.4 디지털 배송 (worker)

```
NEW TX per row:
1. SELECT FOR UPDATE SKIP LOCKED (PENDING)
2. status=PROCESSING UPDATE
COMMIT
———————
외부 호출 (PDFBox + GDrive)         ← 트랜잭션 밖
———————
NEW TX:
3. storage_file_id + status=READY UPDATE
4. publishEvent
COMMIT
```

→ 외부 호출은 **트랜잭션 밖** (DB 락 5분 방지).

---

## 4. AFTER_COMMIT 패턴

```java
@TransactionalEventListener(phase = AFTER_COMMIT)
public void onPaymentApproved(PaymentApproved ev) {
    // 외부 호출 (FCM / 이메일 / GDrive / 워터마크 worker enqueue)
    // 실패해도 결제 트랜잭션은 commit 됨
}
```

→ signup 의 outbox 패턴 동일.

자세히: [[../signup/transactions|↗ signup transactions]].

---

## 5. 함정

### 함정 1 — Redis Lua 안 쓰고 DB UPDATE 만
race → overselling.
→ Redis Lua.

### 함정 2 — 결제 confirm 안 의 PG 호출 timeout
DB 락 30s.
→ short timeout (5s) + webhook 보강.

### 함정 3 — 외부 호출 트랜잭션 안
GDrive 100MB upload → DB 락 30s.
→ AFTER_COMMIT + worker.

### 함정 4 — version optimistic 미사용
lost update.
→ 모든 핵심 도메인에 version.

### 함정 5 — ShedLock 없이 @Scheduled
worker 2번 실행.
→ ShedLock.

---

## 6. 관련

- [[product|↑ hub]]
- [[design-decisions/inventory-strategy]]
- [[design-decisions/payment-flow]]
- [[design-decisions/refund-policy]]
- [[../../distributed-lock|↗ distributed-lock]]
- [[pitfalls/concurrency-pitfalls]]
- [[pitfalls/payment-pitfalls]]
