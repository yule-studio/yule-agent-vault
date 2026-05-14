---
title: "재고 차감 구현 — Redis Lua + DB 동기"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:38:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - implementation
  - inventory
  - concurrency
---

# 재고 차감 구현 — Redis Lua + DB 동기

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[implementation|↑ hub]]**

---

## 1. Redis Lua

```lua
-- decrement_stock.lua
local key = KEYS[1]
local qty = tonumber(ARGV[1])
local current = tonumber(redis.call('GET', key) or '0')
if current < qty then return -1 end
return redis.call('DECRBY', key, qty)
```

---

## 2. Service

```java
@Service
@RequiredArgsConstructor
public class InventoryService {

    private final StringRedisTemplate redis;
    private final InventoryRepository repo;
    private final DefaultRedisScript<Long> decrementScript;
    private final DefaultRedisScript<Long> restoreScript;

    public int decrement(SkuId skuId, int qty) {
        var key = "stock:sku:" + skuId.value();
        Long remaining = redis.execute(decrementScript, List.of(key), String.valueOf(qty));
        if (remaining < 0) throw new InsufficientStockException(skuId);
        return remaining.intValue();
    }

    public void restore(SkuId skuId, int qty) {
        var key = "stock:sku:" + skuId.value();
        redis.opsForValue().increment(key, qty);
    }

    // DB 동기 cron
    @Scheduled(fixedDelay = 15 * 60_000)
    public void syncToDb() {
        var all = repo.findAll();
        for (var inv : all) {
            var redisQty = redis.opsForValue().get("stock:sku:" + inv.skuId().value());
            if (redisQty == null) continue;
            // mismatch 시 alert
            var redisLong = Long.parseLong(redisQty);
            if (Math.abs(redisLong - inv.onHand()) > 0) {
                log.warn("inventory mismatch: sku={} redis={} db={}",
                    inv.skuId(), redisLong, inv.onHand());
                metrics.counter("inventory.mismatch").increment();
            }
        }
    }
}
```

---

## 3. 부족 알림

```java
@TransactionalEventListener(phase = AFTER_COMMIT)
public void onInventoryDecremented(InventoryDecremented ev) {
    if (ev.remaining() <= ev.lowThreshold()) {
        events.publishEvent(new InventoryLow(ev.skuId(), ev.remaining()));
    }
    if (ev.remaining() == 0) {
        productService.markSoldOut(ev.productId());
    }
}
```

---

## 4. 함정

### 함정 1 — Redis 만 (DB X)
Redis 장애 시 재고 손실.
→ DB 동기 cron.

### 함정 2 — Lua 안 씀
race.
→ atomic Lua.

### 함정 3 — restore 동시성
중복 복원.
→ idempotent (order status 검증).

---

## 5. 관련

- [[implementation|↑ hub]]
- [[../design-decisions/inventory-strategy]]
- [[../database/inventory-table]]
- [[../../distributed-lock|↗ distributed-lock]]
