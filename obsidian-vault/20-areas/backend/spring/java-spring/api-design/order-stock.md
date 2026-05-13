---
title: "주문 + 재고 동시성 — Java Spring Boot 레시피"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T14:30:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - recipe
  - commerce
  - order
  - concurrency
---

# 주문 + 재고 동시성 — Java Spring Boot 레시피

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Order Aggregate / state machine / 낙관·비관·분산 락 비교 / saga 첫걸음 |

**[[api-design|↑ api-design hub]]**

> 전제: [[product-crud]] · [[cart]]. `products`, `product_options`, `carts`, `cart_items` 존재.
> **참고**: 쿠팡 (티켓 / 한정수량), 무신사 (드롭 / 선착순), 당근 (단일 거래).

---

## 1. 무엇을 만드는가

```
POST /api/v1/orders                            # 카트 → 주문 생성 (재고 hold + 가격 snapshot)
GET  /api/v1/orders/{id}                       # 주문 조회
POST /api/v1/orders/{id}/cancel                # 결제 전 취소 (재고 복구)
POST /api/v1/orders/{id}/confirm-payment       # 결제 완료 후 확정 (재고 차감 영구)
GET  /api/v1/orders?status=...                 # 내 주문 목록
```

### 1.1 요청 / 응답 — 주문 생성

```http
POST /api/v1/orders
Authorization: Bearer ...
Idempotency-Key: 01HZIDEMP-...                   # 클라이언트가 발급
{
  "items": [
    { "productId": "01HZ...", "optionId": "01HZ...", "quantity": 2 }
  ],
  "shippingAddress": {
    "recipient": "홍길동",
    "phone": "010-0000-0000",
    "zip": "06000",
    "address1": "...",
    "address2": "..."
  },
  "memo": null
}

201 Created
{
  "data": {
    "orderId": "01HZORD...",
    "orderNumber": "20260514-000001",
    "status": "PENDING_PAYMENT",
    "totalKrw": 98000,
    "expiresAt": "2026-05-14T14:45:00Z"          # 15분 내 결제 안 하면 자동 취소
  }
}
```

### 1.2 비기능

- **재고 hold** = 주문 생성 시점에 차감. 결제 후 영구 / 미결제 만료 시 복구.
- **`Idempotency-Key`** — 네트워크 retry 로 중복 주문 차단.
- **가격 / 상품 정보 snapshot** — 주문 생성 후 상품 가격 변경되어도 주문은 그 시점 가격.
- **15분 만료** (`expiresAt`) — Scheduler 가 만료 주문 cancel.
- **트랜잭션 격리** — 같은 옵션에 100명 동시 주문 = 정확히 재고 만큼만 성공.
- **상태 머신** — `PENDING_PAYMENT → PAID → SHIPPED → DELIVERED` / `CANCELED`.

---

## 2. 도메인 모델

### 2.1 Order Aggregate

```java
// src/main/java/com/example/shop/domain/order/Order.java
public final class Order {

    private final OrderId id;
    private final String orderNumber;            // 사람이 보는 번호 — "20260514-000001"
    private final UserId buyerId;
    private final List<OrderItem> items;
    private final Money totalAmount;
    private final ShippingAddress shipping;
    private final String memo;
    private OrderStatus status;
    private final Instant createdAt;
    private Instant updatedAt;
    private Instant expiresAt;
    private final List<DomainEvent> events = new ArrayList<>();

    private Order(OrderId id, String orderNumber, UserId buyerId,
                  List<OrderItem> items, Money total, ShippingAddress shipping, String memo,
                  OrderStatus status, Instant createdAt, Instant updatedAt, Instant expiresAt) {
        this.id = id; this.orderNumber = orderNumber; this.buyerId = buyerId;
        this.items = items; this.totalAmount = total; this.shipping = shipping; this.memo = memo;
        this.status = status; this.createdAt = createdAt;
        this.updatedAt = updatedAt; this.expiresAt = expiresAt;
    }

    public static Order place(OrderId id, String orderNumber, UserId buyerId,
                              List<OrderItem> items, ShippingAddress shipping, String memo,
                              Instant now, Duration paymentTtl) {
        if (items == null || items.isEmpty()) throw new IllegalArgumentException("no items");
        var total = items.stream()
            .map(OrderItem::lineTotal)
            .reduce(Money::add)
            .orElseThrow();
        var order = new Order(
            id, orderNumber, buyerId, List.copyOf(items),
            total, shipping, memo,
            OrderStatus.PENDING_PAYMENT, now, now, now.plus(paymentTtl)
        );
        order.events.add(new OrderPlaced(id, buyerId, total, items, now));
        return order;
    }

    public void confirmPayment(Instant now) {
        if (status != OrderStatus.PENDING_PAYMENT)
            throw new IllegalStateException("cannot confirm from " + status);
        if (now.isAfter(expiresAt))
            throw new OrderExpiredException(id);
        this.status = OrderStatus.PAID;
        this.updatedAt = now;
        events.add(new OrderPaid(id, totalAmount, now));
    }

    public void cancel(String reason, Instant now) {
        if (status != OrderStatus.PENDING_PAYMENT && status != OrderStatus.PAID)
            throw new IllegalStateException("cannot cancel from " + status);
        var wasPaid = (status == OrderStatus.PAID);
        this.status = OrderStatus.CANCELED;
        this.updatedAt = now;
        events.add(new OrderCanceled(id, reason, wasPaid, items, now));
    }

    public void markExpired(Instant now) {
        if (status != OrderStatus.PENDING_PAYMENT)
            throw new IllegalStateException("only PENDING can expire");
        this.status = OrderStatus.EXPIRED;
        this.updatedAt = now;
        events.add(new OrderExpired(id, items, now));
    }

    public void ship(String trackingNumber, Instant now) {
        if (status != OrderStatus.PAID) throw new IllegalStateException("not paid");
        this.status = OrderStatus.SHIPPED;
        this.updatedAt = now;
        events.add(new OrderShipped(id, trackingNumber, now));
    }

    public boolean isExpired(Instant now) {
        return status == OrderStatus.PENDING_PAYMENT && now.isAfter(expiresAt);
    }

    public boolean isOwnedBy(UserId user) { return buyerId.equals(user); }

    // getters omitted for brevity

    public List<DomainEvent> pullDomainEvents() {
        var copy = List.copyOf(events); events.clear(); return copy;
    }
}

public enum OrderStatus {
    PENDING_PAYMENT,    // 재고 hold 됨
    PAID,               // 결제 완료, 재고 영구 차감
    SHIPPED,
    DELIVERED,
    CANCELED,           // 결제 전 취소 / 결제 후 환불
    EXPIRED             // 15분 미결제 → 자동
}
```

### 2.2 OrderItem — snapshot

```java
public final class OrderItem {
    private final OrderItemId id;
    private final ProductId productId;
    private final ProductOptionId optionId;
    private final String productNameSnapshot;       // 주문 시점 상품명
    private final String optionLabelSnapshot;       // "Size: M"
    private final Money unitPriceSnapshot;          // 주문 시점 단가 (base + option extra)
    private final int quantity;

    public OrderItem(OrderItemId id, ProductId productId, ProductOptionId optionId,
                     String productName, String optionLabel, Money unitPrice, int quantity) {
        if (quantity <= 0) throw new IllegalArgumentException("quantity > 0");
        this.id = id; this.productId = productId; this.optionId = optionId;
        this.productNameSnapshot = productName; this.optionLabelSnapshot = optionLabel;
        this.unitPriceSnapshot = unitPrice; this.quantity = quantity;
    }

    public Money lineTotal() { return unitPriceSnapshot.multiply(quantity); }
    // getters
}
```

> **핵심**: 주문 생성 시 가격 / 상품명 / 옵션 라벨을 **그 시점 스냅샷** 으로 보관. 상품 가격이 나중에 변경되어도 주문 내역은 동일.

### 2.3 Domain Events

```java
public record OrderPlaced(OrderId orderId, UserId buyerId, Money total,
                          List<OrderItem> items, Instant occurredAt) implements DomainEvent {}
public record OrderPaid(OrderId orderId, Money total, Instant occurredAt) implements DomainEvent {}
public record OrderCanceled(OrderId orderId, String reason, boolean wasPaid,
                            List<OrderItem> items, Instant occurredAt) implements DomainEvent {}
public record OrderExpired(OrderId orderId, List<OrderItem> items, Instant occurredAt) implements DomainEvent {}
public record OrderShipped(OrderId orderId, String trackingNumber, Instant occurredAt) implements DomainEvent {}
```

### 2.4 Repository

```java
public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(OrderId id);
    Optional<Order> findByIdempotencyKey(String key);
    Page<Order> findByBuyer(UserId buyer, OrderStatus status, Pageable pageable);
    List<Order> findExpiredCandidates(Instant before, int limit);
}
```

---

## 3. 동시성 — 재고 차감의 3가지 방법

### 3.1 비교

| 방식 | 동작 | 처리량 | 락 충돌 | 사용처 |
| --- | --- | --- | --- | --- |
| **낙관 락** (@Version) | UPDATE 시 version 검증, 실패 시 재시도 | 높음 | 충돌 시 retry 비용 | 충돌 적은 일반 상품 |
| **비관 락** (SELECT FOR UPDATE) | DB 행 락 → 한 명씩 처리 | 낮음 (직렬화) | 안전 | 한정 수량 / 특가 |
| **분산 락** (Redisson) | Redis lock 으로 옵션별 직렬화 | 중간 | 안전 | DB 부하 분산 |

### 3.2 낙관 락

```java
@Entity
class ProductOptionJpaEntity {
    @Version
    @Column(nullable = false)
    private long version;

    @Column(nullable = false)
    private int stock;
}

// UseCase
@Transactional
public void decreaseStock(ProductOptionId id, int qty) {
    var opt = optRepo.findById(id.value()).orElseThrow();
    opt.setStock(opt.getStock() - qty);    // UPDATE ... WHERE id=? AND version=?
    // version 다르면 OptimisticLockException
}
```

JPA 가 자동 `WHERE version=?` 추가. 동시 두 트랜잭션 → 한쪽 `OptimisticLockException` → 재시도 (Spring Retry).

```java
@Retryable(
    value = OptimisticLockingFailureException.class,
    maxAttempts = 5,
    backoff = @Backoff(delay = 10, multiplier = 2)    // 10ms, 20, 40, 80, 160
)
@Transactional
public void decreaseStock(ProductOptionId id, int qty) { ... }
```

**적합**: 충돌이 드문 일반 상품. retry 5~10회 안에 성공 가능.

### 3.3 비관 락 — `SELECT FOR UPDATE`

```java
public interface ProductOptionJpaRepository extends JpaRepository<...> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select o from ProductOptionJpaEntity o where o.id = :id")
    Optional<ProductOptionJpaEntity> findByIdForUpdate(@Param("id") String id);
}
```

```sql
-- 실제 SQL
SELECT * FROM product_options WHERE id = '01HZOPT...' FOR UPDATE;
-- 다른 트랜잭션은 같은 행 lock 대기
```

```java
@Transactional
public void decreaseStock(ProductOptionId id, int qty) {
    var opt = optRepo.findByIdForUpdate(id.value()).orElseThrow();
    if (opt.getStock() < qty) throw new InsufficientStockException(id, qty, opt.getStock());
    opt.setStock(opt.getStock() - qty);
}
```

**적합**: 한정수량 / 선착순 / 충돌이 빈번한 경우. 직렬화되니 처리량 낮음. **타임아웃 필수**:

```sql
SET LOCAL lock_timeout = '3s';     -- PG
```

```java
@QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000"))
```

### 3.4 분산 락 — Redisson

```kotlin
// build.gradle.kts
implementation("org.redisson:redisson-spring-boot-starter:3.27.2")
```

```java
@Service
public class StockService {

    private final RedissonClient redisson;
    private final ProductOptionJpaRepository optRepo;
    private final TransactionTemplate tx;

    @SneakyThrows
    public void decreaseStock(ProductOptionId id, int qty) {
        var lock = redisson.getLock("lock:option:" + id.value());
        if (!lock.tryLock(3, 5, TimeUnit.SECONDS))     // wait 3s, hold 5s
            throw new LockAcquireFailedException(id);
        try {
            tx.executeWithoutResult(status -> {
                var opt = optRepo.findById(id.value()).orElseThrow();
                if (opt.getStock() < qty)
                    throw new InsufficientStockException(id, qty, opt.getStock());
                opt.setStock(opt.getStock() - qty);
            });
        } finally {
            lock.unlock();
        }
    }
}
```

**적합**: DB 락 부하 분산. 단일 서버는 굳이 분산 락 필요 없음.

> **함정**: 분산 락 hold 시간 < 락 TTL. 트랜잭션이 길면 락 만료 후 다른 노드가 락 획득 → 동시 수정. **fencing token** (Redlock 한계 — Martin Kleppmann).

### 3.5 어떤 방법을 언제

```
질문 1: 트래픽 / 충돌 빈도?
├─ 낮음 (일반 상품, 동시 같은 옵션 < 10 TPS) → 낙관 락
└─ 높음 →

질문 2: 단일 DB 인가?
├─ Yes → 비관 락 (SELECT FOR UPDATE)
└─ No (sharded / 여러 서비스) → 분산 락 + DB UPDATE

질문 3: 극단 한정수량 (선착순 1000개) 인가?
├─ Yes → Redis 카운터 decr 만 + 비동기 DB 동기화
└─ No  → 위 답
```

---

## 4. DB 스키마

```sql
-- V40__create_orders.sql
CREATE TABLE orders (
  id                CHAR(26) PRIMARY KEY,
  order_number      VARCHAR(30) NOT NULL,
  buyer_id          CHAR(26) NOT NULL REFERENCES users(id),
  total_amount_krw  BIGINT NOT NULL CHECK (total_amount_krw >= 0),
  shipping_recipient VARCHAR(50) NOT NULL,
  shipping_phone    VARCHAR(20) NOT NULL,
  shipping_zip      VARCHAR(10) NOT NULL,
  shipping_addr1    VARCHAR(200) NOT NULL,
  shipping_addr2    VARCHAR(200),
  memo              VARCHAR(500),
  status            VARCHAR(20) NOT NULL,
  version           BIGINT NOT NULL DEFAULT 0,
  idempotency_key   VARCHAR(50),
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at        TIMESTAMPTZ
);
CREATE UNIQUE INDEX ux_orders_order_number ON orders (order_number);
CREATE UNIQUE INDEX ux_orders_idempotency
  ON orders (buyer_id, idempotency_key) WHERE idempotency_key IS NOT NULL;
CREATE INDEX ix_orders_buyer_status_created ON orders (buyer_id, status, created_at DESC);
CREATE INDEX ix_orders_expires
  ON orders (status, expires_at) WHERE status = 'PENDING_PAYMENT';

-- V41__create_order_items.sql
CREATE TABLE order_items (
  id                      CHAR(26) PRIMARY KEY,
  order_id                CHAR(26) NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id              CHAR(26) NOT NULL REFERENCES products(id),
  option_id               CHAR(26) NOT NULL REFERENCES product_options(id),
  product_name_snapshot   VARCHAR(200) NOT NULL,
  option_label_snapshot   VARCHAR(200) NOT NULL,
  unit_price_krw_snapshot BIGINT NOT NULL CHECK (unit_price_krw_snapshot >= 0),
  quantity                INTEGER NOT NULL CHECK (quantity > 0)
);
CREATE INDEX ix_order_items_order ON order_items (order_id);
CREATE INDEX ix_order_items_product ON order_items (product_id);

-- V42__create_stock_movements.sql — 재고 변동 감사 로그
CREATE TABLE stock_movements (
  id           CHAR(26) PRIMARY KEY,
  option_id    CHAR(26) NOT NULL REFERENCES product_options(id),
  order_id     CHAR(26) REFERENCES orders(id),
  delta        INTEGER NOT NULL,           -- 음수 = 차감, 양수 = 복구
  reason       VARCHAR(50) NOT NULL,        -- ORDER_PLACED / ORDER_CANCELED / ORDER_EXPIRED
  occurred_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ix_stock_movements_option ON stock_movements (option_id, occurred_at DESC);
```

> **함정**: `stock_movements` 같은 감사 로그가 없으면 "왜 재고가 줄었지?" 추적 불가. **모든 재고 변동을 이벤트 row 로 기록**.

---

## 5. PlaceOrderUseCase (주문 생성 — 가장 중요한 트랜잭션)

```java
// src/main/java/com/example/shop/application/order/PlaceOrderUseCase.java
@Service
public class PlaceOrderUseCase {

    private final OrderRepository orders;
    private final ProductRepository products;
    private final ProductOptionJpaRepository optionRepo;
    private final OrderNumberGenerator orderNumbers;
    private final IdGenerator ids;
    private final Clock clock;
    private final Duration paymentTtl;
    private final ApplicationEventPublisher events;

    public PlaceOrderUseCase(OrderRepository orders, ProductRepository products,
                             ProductOptionJpaRepository optionRepo,
                             OrderNumberGenerator orderNumbers, IdGenerator ids,
                             Clock clock,
                             @Value("${app.order.payment-ttl}") Duration paymentTtl,
                             ApplicationEventPublisher events) {
        this.orders = orders; this.products = products; this.optionRepo = optionRepo;
        this.orderNumbers = orderNumbers; this.ids = ids;
        this.clock = clock; this.paymentTtl = paymentTtl; this.events = events;
    }

    @Transactional
    public OrderId handle(PlaceOrderCommand cmd) {
        // 1. 멱등성 체크
        if (cmd.idempotencyKey() != null) {
            var existing = orders.findByIdempotencyKey(cmd.idempotencyKey());
            if (existing.isPresent()) return existing.get().id();
        }

        // 2. 옵션 ID 정렬 — 데드락 방지 (항상 같은 순서로 락)
        var sortedItems = cmd.items().stream()
            .sorted(Comparator.comparing(i -> i.optionId().value()))
            .toList();

        var now = Instant.now(clock);
        var orderItems = new ArrayList<OrderItem>();

        // 3. 각 옵션에 대해 락 + 재고 차감 + snapshot 생성
        for (var ci : sortedItems) {
            var optEntity = optionRepo.findByIdForUpdate(ci.optionId().value())
                .orElseThrow(() -> new NotFoundException("option"));

            if (optEntity.getStock() < ci.quantity())
                throw new InsufficientStockException(
                    ci.optionId(), ci.quantity(), optEntity.getStock());

            optEntity.setStock(optEntity.getStock() - ci.quantity());

            // 상품 검증 + snapshot 만들기 (가격은 product.basePrice + option.extraPrice)
            var product = products.findById(ci.productId())
                .orElseThrow(() -> new NotFoundException("product"));
            if (product.status() != ProductStatus.ACTIVE)
                throw new ProductNotAvailableException(product.status());

            var unitPrice = product.basePrice().add(Money.krw(optEntity.getExtraPriceKrw()));
            orderItems.add(new OrderItem(
                new OrderItemId(ids.next()),
                ci.productId(), ci.optionId(),
                product.name(),
                optEntity.getName() + ": " + optEntity.getValue(),
                unitPrice,
                ci.quantity()
            ));

            // 감사 로그
            recordStockMovement(ci.optionId(), null, -ci.quantity(), "ORDER_PLACED", now);
        }

        // 4. Order 생성
        var orderId = new OrderId(ids.next());
        var order = Order.place(
            orderId,
            orderNumbers.next(),
            cmd.buyerId(),
            orderItems,
            cmd.shipping(),
            cmd.memo(),
            now,
            paymentTtl
        );
        // idempotency key 도 entity 매핑 시 저장 — adapter 에서 처리

        var saved = orders.save(order);
        saved.pullDomainEvents().forEach(events::publishEvent);
        return saved.id();
    }

    private void recordStockMovement(ProductOptionId optId, OrderId orderId,
                                     int delta, String reason, Instant now) {
        // stock_movements 에 insert — 별도 repo
    }
}
```

> **핵심 포인트**:
> 1. **`Idempotency-Key`** 로 중복 주문 차단 — DB unique 가 진실의 원천
> 2. **옵션 ID 정렬** — 같은 옵션 동시 잠금 시 데드락 방지
> 3. **`SELECT FOR UPDATE`** 비관 락
> 4. **재고 검증 → 차감 → snapshot** 이 한 트랜잭션
> 5. **이벤트 발행** — 결제 / 알림 / 감사

### 5.1 데드락 회피 — 정렬 순서

```
주문 A: option-X, option-Y 둘 다 차감 — 락 순서 X → Y
주문 B: option-Y, option-X 둘 다 차감 — 락 순서 Y → X
→ 데드락
```

해결: **모든 트랜잭션이 같은 순서 (예: id 오름차순) 로 락**. Spring TX 가 자동으로 안 함 → 코드에서 정렬.

---

## 6. 결제 확인 / 취소 / 만료 처리

### 6.1 ConfirmPaymentUseCase — webhook 또는 결제 PG 콜백

```java
@Service
public class ConfirmPaymentUseCase {

    private final OrderRepository orders;
    private final Clock clock;
    private final ApplicationEventPublisher events;

    @Transactional
    public void handle(OrderId id) {
        var order = orders.findById(id).orElseThrow(NotFoundException::new);
        order.confirmPayment(Instant.now(clock));
        orders.save(order);
        order.pullDomainEvents().forEach(events::publishEvent);
    }
}
```

### 6.2 CancelOrderUseCase — 사용자가 취소 (결제 전)

```java
@Service
public class CancelOrderUseCase {

    private final OrderRepository orders;
    private final ProductOptionJpaRepository optionRepo;
    private final Clock clock;
    private final ApplicationEventPublisher events;

    @Transactional
    public void handle(OrderId id, UserId buyer, String reason) {
        var order = orders.findById(id).orElseThrow(NotFoundException::new);
        if (!order.isOwnedBy(buyer)) throw new ForbiddenException();

        order.cancel(reason, Instant.now(clock));
        orders.save(order);

        // 재고 복구 — 정렬 후 락
        var items = order.items().stream()
            .sorted(Comparator.comparing(i -> i.optionId().value()))
            .toList();
        for (var item : items) {
            var opt = optionRepo.findByIdForUpdate(item.optionId().value()).orElseThrow();
            opt.setStock(opt.getStock() + item.quantity());
            // 감사 로그
        }

        order.pullDomainEvents().forEach(events::publishEvent);
    }
}
```

### 6.3 ExpireOrdersJob — 15분 미결제 자동 만료

```java
@Component
public class ExpireOrdersJob {

    private final OrderRepository orders;
    private final ProductOptionJpaRepository optionRepo;
    private final Clock clock;

    @Scheduled(fixedDelay = 60_000)        // 매 분
    public void run() {
        var now = Instant.now(clock);
        // ShedLock 으로 다중 인스턴스 동시 실행 방지
        var candidates = orders.findExpiredCandidates(now, 100);
        for (var order : candidates) {
            expireOne(order.id());
        }
    }

    @Transactional
    public void expireOne(OrderId id) {
        var order = orders.findById(id).orElseThrow();
        if (!order.isExpired(Instant.now(clock))) return;
        order.markExpired(Instant.now(clock));
        orders.save(order);
        // 재고 복구 — 정렬 후 락 (위 cancel 과 동일)
    }
}
```

> **함정**: 다중 인스턴스에서 `@Scheduled` 동시 실행. **ShedLock** 으로 단일 실행 보장:

```java
@SchedulerLock(name = "expireOrders", lockAtMostFor = "5m", lockAtLeastFor = "30s")
```

---

## 7. Idempotency-Key 처리

### 7.1 DB unique 가 진실의 원천

```sql
CREATE UNIQUE INDEX ux_orders_idempotency
  ON orders (buyer_id, idempotency_key) WHERE idempotency_key IS NOT NULL;
```

### 7.2 동작

```
클라이언트 → 주문 요청 (Idempotency-Key: K)
서버:
  1. orders WHERE buyer=u AND idempotency_key=K 조회
  2. 있으면 → 기존 주문 반환 (200 OK, 같은 응답)
  3. 없으면 → 주문 생성 + idempotency_key 저장
  4. 네트워크 retry 로 같은 K 가 또 옴 → DataIntegrityViolationException
     → 기다렸다가 다시 조회 → 기존 주문 반환
```

### 7.3 키 관리

- 키 형식: `ULID` 또는 `UUID` (클라이언트 발급)
- TTL: 24시간 (그 이후는 unique 충돌 무관 — 다른 주문)
- 만료된 키 청소: `DELETE FROM orders WHERE created_at < now() - INTERVAL '30d' AND status IN ('PAID', 'CANCELED', 'EXPIRED')` (X — 주문은 삭제 X) → 별도 idempotency 테이블이 더 깔끔

```sql
CREATE TABLE idempotency_keys (
  key         VARCHAR(50) PRIMARY KEY,
  buyer_id    CHAR(26) NOT NULL,
  resource_id CHAR(26) NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at  TIMESTAMPTZ NOT NULL
);
```

---

## 8. Controller

```java
@RestController
@RequestMapping("/api/v1/orders")
@PreAuthorize("isAuthenticated()")
public class OrderController {

    private final PlaceOrderUseCase place;
    private final CancelOrderUseCase cancel;
    private final ConfirmPaymentUseCase confirm;
    private final GetOrderUseCase getOne;

    public OrderController(PlaceOrderUseCase place, CancelOrderUseCase cancel,
                           ConfirmPaymentUseCase confirm, GetOrderUseCase getOne) {
        this.place = place; this.cancel = cancel;
        this.confirm = confirm; this.getOne = getOne;
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ApiResponse<OrderCreatedResponse> create(
        @Valid @RequestBody PlaceOrderRequest req,
        @RequestHeader(value = "Idempotency-Key", required = false) String idempotencyKey,
        Authentication auth
    ) {
        var cmd = toCommand(req, new UserId(auth.getName()), idempotencyKey);
        var id = place.handle(cmd);
        var order = getOne.handle(id, new UserId(auth.getName()));
        return ApiResponse.ok(new OrderCreatedResponse(
            order.id(), order.orderNumber(), order.status().name(),
            order.totalAmount().amountMinorUnit(),
            order.expiresAt()
        ));
    }

    @PostMapping("/{id}/cancel")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void cancel(@PathVariable String id, @RequestBody CancelOrderRequest req,
                       Authentication auth) {
        cancel.handle(new OrderId(id), new UserId(auth.getName()), req.reason());
    }

    @GetMapping("/{id}")
    public ApiResponse<OrderDetailResponse> get(@PathVariable String id, Authentication auth) {
        return ApiResponse.ok(getOne.detailFor(new OrderId(id), new UserId(auth.getName())));
    }
}

public record PlaceOrderRequest(
    @Valid @NotEmpty List<ItemDto> items,
    @Valid @NotNull ShippingDto shipping,
    @Size(max = 500) String memo
) {
    public record ItemDto(
        @NotBlank String productId,
        @NotBlank String optionId,
        @Min(1) @Max(99) int quantity
    ) {}
    public record ShippingDto(
        @NotBlank String recipient, @NotBlank String phone,
        @NotBlank String zip, @NotBlank String address1, String address2
    ) {}
}
```

---

## 9. Saga 의 첫 걸음 — 결제 실패 시 재고 복구

이 레시피의 **단일 DB** 모델에선 `@Transactional` 한 번이면 충분. MSA 로 분리되면:

```
[Order Service] PlaceOrder
   ├─ orders.save (Order, PENDING_PAYMENT)
   └─ event: OrderPlaced (Kafka)
              ↓
       [Stock Service] reserve stock — 성공 시 OrderStockReserved 이벤트
              ↓
       [Payment Service] charge — 실패 시 OrderPaymentFailed 이벤트
              ↓
       [Stock Service] release stock — OrderStockReleased
              ↓
       [Order Service] order.cancel()
```

→ Saga / Compensation. 본 레시피 범위 밖. `saga-pattern` (다음).

---

## 10. 테스트

### 10.1 동시성 — 100명이 재고 50개에 주문

```java
@Test
void at_most_50_orders_succeed_when_stock_is_50_and_100_concurrent_requests() throws Exception {
    var opt = optionFixture.createWithStock(50);
    var users = userFixture.createMany(100);

    var executor = Executors.newFixedThreadPool(20);
    var successes = new AtomicInteger();
    var fails = new AtomicInteger();
    var latch = new CountDownLatch(100);

    for (var u : users) {
        executor.submit(() -> {
            try {
                var auth = "Bearer " + tokens.access(u);
                var res = rest.exchange("/api/v1/orders", HttpMethod.POST,
                    new HttpEntity<>(orderBody(opt), headers(auth)), Map.class);
                if (res.getStatusCode().value() == 201) successes.incrementAndGet();
                else fails.incrementAndGet();
            } finally { latch.countDown(); }
        });
    }
    latch.await(30, TimeUnit.SECONDS);
    executor.shutdown();

    assertThat(successes.get()).isEqualTo(50);
    assertThat(fails.get()).isEqualTo(50);

    var optReloaded = optionFixture.reload(opt);
    assertThat(optReloaded.getStock()).isEqualTo(0);
}
```

### 10.2 Idempotency

```java
@Test
void same_idempotency_key_returns_same_order() {
    var key = "test-key-1";
    var h = headersWithBearer().with("Idempotency-Key", key);

    var r1 = rest.exchange("/api/v1/orders", HttpMethod.POST,
        new HttpEntity<>(body(), h), Map.class);
    var r2 = rest.exchange("/api/v1/orders", HttpMethod.POST,
        new HttpEntity<>(body(), h), Map.class);

    var id1 = orderIdFrom(r1);
    var id2 = orderIdFrom(r2);
    assertThat(id1).isEqualTo(id2);
}
```

### 10.3 만료

```java
@Test
void unpaid_order_expires_and_stock_is_restored_after_15m() {
    var opt = optionFixture.createWithStock(10);
    var order = orderFixture.place(opt, quantity = 3);
    assertThat(optionFixture.reload(opt).getStock()).isEqualTo(7);

    // 시간 진행
    clockFixture.advance(Duration.ofMinutes(16));
    expireOrdersJob.run();

    assertThat(optionFixture.reload(opt).getStock()).isEqualTo(10);    // 복구
}
```

---

## 11. 운영 체크리스트

- [ ] `Idempotency-Key` 검증 + DB unique
- [ ] 옵션 ID 정렬로 데드락 방지
- [ ] `lock_timeout = 3s` 명시 (무한 대기 방지)
- [ ] `@Scheduled` 만료 job 에 ShedLock
- [ ] 재고 변동 매번 `stock_movements` 기록
- [ ] 상태 전이 위반 시 명확한 예외 + 알림
- [ ] 결제 webhook 의 retry 멱등성 (`order.confirmPayment` 가 중복 호출 안전)
- [ ] 주문 번호는 사람이 보는 거 (orderNumber) + 시스템 ID (orderId) 분리
- [ ] 주문 만료 TTL 설정 (`app.order.payment-ttl: PT15M`)
- [ ] 비관 락 vs 낙관 락 결정 — 트래픽 측정 후 선택
- [ ] APM 으로 `decreaseStock` latency / lock wait 모니터

---

## 12. 함정 모음

### 함정 1 — Idempotency-Key 미적용
네트워크 retry 로 중복 주문. 사용자 결제 두 번. **헤더 강제 + DB unique**.

### 함정 2 — 데드락
옵션 A, B 동시 주문 시 잠금 순서 다르면 데드락. **항상 ID 정렬 후 락**.

### 함정 3 — 재고 검증과 차감을 따로
```java
if (opt.stock >= qty) {           // 1. 검증
    // 다른 트랜잭션이 차감 (race)
    opt.stock -= qty;             // 2. 차감 → 음수 가능
}
```
**한 트랜잭션 + FOR UPDATE 또는 낙관 락**.

### 함정 4 — `@Scheduled` 다중 인스턴스 동시 실행
같은 주문 두 번 만료 처리. **ShedLock**.

### 함정 5 — 주문 가격을 product 에서 매번 조회
주문 내역에서 가격이 바뀜. **OrderItem 에 snapshot 필수**.

### 함정 6 — 결제 webhook retry 무방비
PG 가 webhook 을 5번 retry → confirmPayment 5번 호출 → 상태 전이 예외 또는 중복 처리. **`order.confirmPayment` 가 PAID 상태에 멱등** (이미 PAID 면 no-op).

```java
public void confirmPayment(Instant now) {
    if (status == OrderStatus.PAID) return;     // 멱등
    if (status != OrderStatus.PENDING_PAYMENT)
        throw new IllegalStateException(...);
    ...
}
```

### 함정 7 — 락 hold 시간 길게
주문 트랜잭션 안에서 외부 API (배송 / 알림) 호출 → 락 hold + 다른 주문 대기. **외부 호출은 트랜잭션 밖** (AFTER_COMMIT).

### 함정 8 — 비관 락 + 작은 옵션 = 처리량 절벽
인기 상품 1개 옵션에 락 = 줄 서기. **분산 락 또는 Redis 카운터** 로 부하 분산.

### 함정 9 — 만료 처리 누락
TTL 만 두고 실제 cancel job 없음 → 영원히 PENDING. **반드시 job + 모니터**.

### 함정 10 — Order 가 Cart 를 직접 의존
Order 가 Cart 의 도메인을 import = 결합. **Order 는 ItemSnapshot 만 받고 Cart 모름**. Application Layer 가 Cart → Order 변환.

---

## 13. 관련

- [[product-crud]] · [[cart]]
- [[payment-pg]] (다음) — 주문 → 결제
- [[../pitfalls/concurrency-pitfalls]] (작성 예정) — 낙관/비관/분산 깊이
- [[../pitfalls/transaction-pitfalls]] (작성 예정) — 락 hold / outside session
- [[../../../../architecture/architecture|↗ architecture]] — saga 까지 가는 길
- [[api-design|↑ api-design hub]]
