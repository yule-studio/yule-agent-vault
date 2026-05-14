---
title: "재고 차감 전략 — 주문 시점 + Redis atomic"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T16:55:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - design-decisions
  - inventory
  - concurrency
---

# 재고 차감 전략 — 주문 시점 + Redis atomic

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[design-decisions|↑ design-decisions hub]]**

> "언제 재고 차감 / 동시 100 주문 1 재고 시 어떻게" — 결제 사고의 1순위.

---

## 1. 본 vault 결정

| 시점 | 처리 |
| --- | --- |
| 주문 생성 시점 | **재고 차감 (Redis atomic Lua script)** |
| 결제 실패 / timeout 30m | **재고 복원** (compensating action) |
| 환불 | **재고 복원** (실물만, 디지털 X) |

---

## 2. 왜 / 안 하면 / 대안 / 트레이드오프

### 2.1 왜 주문 시점 (결제 후 X)

- 결제 시점은 PG 외부 의존 — 결제 시도하다 재고 없으면 사용자 confusion.
- 주문 시점 차감 = 결제 보장 = 사용자 신뢰.

### 2.2 안 하면

| 잘못 | 사고 |
| --- | --- |
| 결제 후 차감 | 결제 완료된 사용자 N 명, 재고 1 개 — overselling |
| DB UPDATE 의존 (락 X) | 동시 100 요청 → 100 차감 (재고 1 인데) |
| Redis atomic 만 (DB 의 일관성 X) | Redis 장애 시 inconsistency |
| 복원 없음 | 결제 실패 시 재고 영영 잠김 |

### 2.3 대안

| 모델 | 적용 |
| --- | --- |
| **주문 시점 + Redis Lua + DB UPSERT** ★ | 본 vault — 100 TPS |
| DB UPDATE `WHERE stock >= ?` + version | 단순 |
| `FOR UPDATE` (pessimistic) | 안정 but 느림 |
| Optimistic version + retry | scale ↑ |
| Saga / event-sourced | event store + projection |
| Queue (모든 주문을 queue 에) | 직렬화 — 안정 but 느림 |

### 2.4 트레이드오프

- Redis atomic = 빠름 + Redis 장애 위험.
- DB UPSERT 만 = 안정 + 느림 (락 contention).
- → 본 vault: **Redis primary + DB 동기 reconcile cron**.

---

## 3. Redis Lua 코드

```lua
-- decrement_stock.lua
local key = KEYS[1]                  -- stock:productOption:{skuId}
local qty = tonumber(ARGV[1])
local current = tonumber(redis.call('GET', key) or '0')
if current < qty then
  return -1                          -- not enough
end
redis.call('DECRBY', key, qty)
return current - qty
```

```java
@Service
public class InventoryService {

    public int decrement(SkuId skuId, int qty) {
        var script = redisScripts.decrementStock();
        Long remaining = redis.execute(script,
            List.of("stock:sku:" + skuId.value()),
            String.valueOf(qty));
        if (remaining < 0)
            throw new InsufficientStockException(skuId);
        return remaining.intValue();
    }

    public void restore(SkuId skuId, int qty) {
        redis.opsForValue().increment("stock:sku:" + skuId.value(), qty);
    }
}
```

### 3.1 왜 Lua

- Redis 의 multi-command atomic 보장.
- INCR + GET 이 single round trip.

자세히: [[../implementation/inventory-impl]] · [[../../distributed-lock|↗ distributed-lock]].

---

## 4. DB 동기 (eventual)

```sql
CREATE TABLE inventory (
    sku_id       CHAR(26) PRIMARY KEY,
    on_hand      INTEGER NOT NULL CHECK (on_hand >= 0),
    reserved     INTEGER NOT NULL DEFAULT 0,
    version      BIGINT NOT NULL DEFAULT 0,
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

- Redis = realtime (주문 차감).
- DB = source of truth + Redis 와 cron sync (15분).
- Redis fail 시 → DB 의 `on_hand - reserved` 로 fallback.

자세히: [[../database/inventory-table]].

---

## 5. 결제 실패 시 복원

```java
@Component
public class OrderTimeoutWorker {

    @Scheduled(fixedDelay = 60_000)   // 1분
    public void scanExpired() {
        var expired = orders.findExpiredPending(Duration.ofMinutes(30));
        for (var order : expired) {
            order.cancel("TIMEOUT");
            for (var item : order.items())
                inventory.restore(item.skuId(), item.quantity());
            orders.save(order);
        }
    }
}
```

---

## 6. 함정

### 함정 1 — DB UPDATE WHERE stock >= ? 만 (락 X)
동시 100 → MySQL READ_COMMITTED 시 race.
→ Redis Lua 또는 SELECT FOR UPDATE.

### 함정 2 — Redis 만 (DB X)
Redis 장애 시 → 재고 정보 영구 손실.
→ DB 동기 cron.

### 함정 3 — 결제 실패 시 복원 X
재고 영구 잠김.
→ scanExpired worker.

### 함정 4 — 복원 동시성 (환불 + worker)
같은 order 의 복원 2회.
→ idempotent (order.status = CANCELED 검증).

### 함정 5 — 옵션 (SKU) 별 재고 분리 X
red XL 재고 0인데 red M 으로 결제됨.
→ SKU 단위 inventory row.

자세히: [[../pitfalls/concurrency-pitfalls]].

---

## 7. 다른 컨텍스트

### 7.1 무신사 / 쿠팡
SKU 폭증 (red XL / blue M ...) → option 의 카르테시안 곱 → Redis key 폭증 → bitmap / set 으로 압축.

### 7.2 항공권 / 호텔
재고 = unique (1 좌석 1 예약) → optimistic lock.

### 7.3 디지털 (무한 재고)
재고 차감 X — 결제만.

---

## 8. 관련

- [[design-decisions|↑ hub]]
- [[../transactions]] — race condition 분석
- [[../implementation/inventory-impl]]
- [[../../distributed-lock|↗ distributed-lock]]
- [[../pitfalls/concurrency-pitfalls]]
