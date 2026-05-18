---
title: "DDD tactical patterns — Entity / VO / Aggregate / Repository / Domain Service / Event"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-18T21:00:00+09:00
tags: [computer-science, oop, ddd, domain-modeling, tactical-patterns]
home_hub: object-oriented-programming
related:
  - "[[object-oriented-programming]]"
  - "[[concepts]]"
  - "[[value-objects]]"
  - "[[oop-for-backend]]"
  - "[[../software-engineering/software-engineering]]"
---

# DDD tactical patterns — Entity / VO / Aggregate / Repository / Domain Service / Event

**[[object-oriented-programming|↑ object-oriented-programming]]**

---

## 1. 목적

본 문서는 Domain-Driven Design (Eric Evans, 2003) 의 **tactical pattern** 7 가지를 OOP 의 도메인 모델링 도구로 정의한다.

본 문서가 정의하는 것:
- Entity / Value Object / Aggregate / Repository / Domain Service / Domain Event / Factory 의 정의 / 책임 / 관계
- 각 pattern 의 식별 기준 (언제 Entity vs VO vs Service)
- Aggregate 의 일관성 경계 정의 및 트랜잭션 단위
- 백엔드 예제 (주문 / 결제 / 사용자) 의 모델링
- DDD tactical pattern 과 ORM / DI 의 결합

본 문서가 정의하지 않는 것:
- DDD strategic pattern (Bounded Context / Context Map / Ubiquitous Language) — [[../software-engineering/software-engineering]] §5.4
- CQRS / Event Sourcing — [[../distributed-systems/distributed-systems]]
- 4 pillar — [[concepts]]

---

## 2. 범위

| 구분 | 포함 |
| --- | --- |
| 대상 | OOP 백엔드 도메인 모델 |
| 단위 | Entity / VO / Aggregate / Repository / Domain Service / Domain Event / Factory |
| 제외 | strategic pattern (큰 모델 분리) |
| 제외 | 분산 처리 (Saga / Outbox) — [[../distributed-systems/distributed-systems]] |

---

## 3. 용어

| 용어 | 정의 |
| --- | --- |
| **Entity** | identity (식별자) 로 구분되는 도메인 객체. 같은 값이어도 식별자가 다르면 다른 객체. |
| **Value Object** | identity 없이 value 로 구분되는 immutable 객체. |
| **Aggregate** | 일관성 경계를 공유하는 Entity / VO 의 묶음. 외부 참조는 **Aggregate Root** 만 통해. |
| **Aggregate Root** | Aggregate 의 외부 진입점. 모든 외부 호출이 이를 경유. |
| **Repository** | Aggregate 의 영속화를 추상화한 컬렉션-유사 인터페이스. |
| **Domain Service** | 한 Entity / VO 에 속하지 않는 도메인 로직을 담는 stateless 객체. |
| **Domain Event** | 도메인에서 발생한 의미 있는 사실. 보통 past tense (`OrderPlaced`). |
| **Factory** | 복잡한 Aggregate 생성을 캡슐화한 메커니즘. |
| **Bounded Context** | 한 도메인 모델이 유효한 명시적 경계. (strategic pattern) |
| **Ubiquitous Language** | 도메인 전문가와 개발자가 공유하는 어휘. (strategic pattern) |

---

## 4. Entity

### 4.1 정의

identity 로 구분되는 도메인 객체. lifetime 동안 상태가 변하지만 identity 는 변하지 않는다.

### 4.2 식별

| 질문 | 답 → Entity |
| --- | --- |
| 같은 값이어도 다른 객체로 구분되어야 하는가? | 그렇다 → Entity |
| 시간이 지나며 상태가 변하는가? | 그렇다 → Entity |
| 변경 이력을 추적해야 하는가? | 그렇다 → Entity |
| DB 에 row 단위로 저장되는가? | 보통 → Entity |

### 4.3 구현

```java
class Order {
    private final OrderId id;            // identity — final
    private final CustomerId customerId; // immutable reference
    private List<OrderItem> items;       // mutable state
    private OrderStatus status;
    private Money totalPrice;

    // factory method — Aggregate Root 생성 진입점
    static Order create(CustomerId customerId, List<OrderItem> items) {
        if (items.isEmpty()) throw new EmptyOrderException();
        return new Order(OrderId.generate(), customerId, items);
    }

    void cancel() {
        if (status == OrderStatus.SHIPPED) {
            throw new IllegalStateException("cannot cancel shipped order");
        }
        status = OrderStatus.CANCELLED;
    }

    @Override
    public boolean equals(Object o) {
        return o instanceof Order other && id.equals(other.id);
    }

    @Override
    public int hashCode() { return id.hashCode(); }
}
```

| 핵심 | 결과 |
| --- | --- |
| equals/hashCode 는 id 기준 | identity 기반 동등성 |
| state 변경은 의도된 method 만 (`cancel()`) | tell, don't ask |
| invariant 는 method 안 검증 | 캡슐화 |

---

## 5. Value Object

상세: [[value-objects]].

본 영역에서 Entity 와의 위치 차이만 정리.

| 항목 | Entity | Value Object |
| --- | --- | --- |
| identity | id | 없음 |
| 동등성 | id 비교 | 모든 field 비교 |
| 가변성 | mutable | immutable |
| 예 | Order, Customer | Money, Email, Address |
| Aggregate 안 | Root 또는 child Entity | child VO |

---

## 6. Aggregate

### 6.1 정의

일관성 경계를 공유하는 Entity / VO 의 묶음. **외부 참조는 Aggregate Root 만 통한다.**

### 6.2 일관성 경계

| 보장 | 의미 |
| --- | --- |
| Aggregate 안의 invariant 는 트랜잭션 내 항상 일관 | save 시점에 모두 정합 |
| 한 트랜잭션 = 한 Aggregate 만 수정 권장 (DDD 원칙) | 동시 수정 충돌 / 트랜잭션 비대화 회피 |
| 다른 Aggregate 와의 일관성은 eventual | Domain Event / Saga 로 처리 |

### 6.3 구조

```
Aggregate: Order
├── OrderId (identity, VO)
├── CustomerId (참조 — 외부 Aggregate)
├── List<OrderItem> (child entity 또는 VO)
└── Money totalPrice (VO)

외부는 Order 만 참조. OrderItem 은 Order.items() 로만 접근.
```

### 6.4 규칙

| 규칙 | 의미 |
| --- | --- |
| 1. 외부 참조는 Aggregate Root 만 | OrderItem 을 직접 가지면 안 됨 |
| 2. 다른 Aggregate 는 id 로만 참조 | `Customer` 객체가 아니라 `CustomerId` |
| 3. Aggregate 간 일관성은 eventual | 즉시 트랜잭션 X — Domain Event |
| 4. 한 트랜잭션에 한 Aggregate 수정 | Vernon 권장 (예외: 같은 BC 의 작은 Aggregate) |
| 5. Repository 도 Aggregate 단위 | `OrderRepository` (Order Aggregate) / `OrderItemRepository` 없음 |

### 6.5 Aggregate 크기 결정

| 크기 | 트레이드오프 |
| --- | --- |
| 너무 크면 | 트랜잭션 길어짐 + 동시 수정 충돌 ↑ |
| 너무 작으면 | invariant 가 여러 Aggregate 에 흩어짐 → eventual 일관성 부담 |
| 가이드 | invariant 가 즉시 일관성 요구 → 같은 Aggregate. 그렇지 않으면 분리. |

### 6.6 예 — Order Aggregate

```java
class Order {                     // Aggregate Root
    private final OrderId id;
    private final CustomerId customerId;       // 다른 Aggregate 는 id 로만 참조
    private final List<OrderItem> items;       // child — 외부 직접 접근 X
    private OrderStatus status;
    private Money totalPrice;

    void addItem(Product product, Quantity qty) {     // Root 통해서만
        if (status != OrderStatus.DRAFT) {
            throw new CannotModifyAfterSubmitException();
        }
        items.add(new OrderItem(product.id(), product.price(), qty));
        recalculateTotal();
    }

    void place() {
        if (items.isEmpty()) throw new EmptyOrderException();
        status = OrderStatus.PLACED;
    }

    private void recalculateTotal() {
        totalPrice = items.stream()
            .map(OrderItem::subtotal)
            .reduce(Money.ZERO, Money::plus);
    }
}

class OrderItem {                 // child Entity (또는 VO)
    private final ProductId productId;
    private final Money unitPrice;
    private final Quantity quantity;

    Money subtotal() { return unitPrice.multiply(quantity.value()); }
}
```

---

## 7. Repository

### 7.1 정의

Aggregate 의 영속화를 컬렉션처럼 추상화하는 인터페이스. 도메인 계층이 정의하고, infrastructure 계층이 구현한다.

### 7.2 인터페이스

```java
// 도메인 계층
interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
    List<Order> findByCustomer(CustomerId customerId);
}
```

### 7.3 구현 (infrastructure 계층)

```java
@Repository
class JpaOrderRepository implements OrderRepository {
    private final OrderJpaEntityRepository jpaRepo;

    public void save(Order order) {
        OrderJpaEntity entity = OrderJpaEntity.from(order);
        jpaRepo.save(entity);
    }

    public Optional<Order> findById(OrderId id) {
        return jpaRepo.findById(id.value()).map(OrderJpaEntity::toDomain);
    }

    public List<Order> findByCustomer(CustomerId customerId) {
        return jpaRepo.findByCustomerId(customerId.value()).stream()
            .map(OrderJpaEntity::toDomain).toList();
    }
}
```

### 7.4 Repository 가 아닌 것

| 패턴 | Repository 와 차이 |
| --- | --- |
| DAO | row 단위 / SQL 노출 / 도메인 무지 |
| ORM Entity 자체 (active record) | 객체가 자기 save 호출 — 도메인과 영속 결합 |
| Query Object | 단순 조회 — Aggregate 무관 (CQRS 의 read side) |

### 7.5 Repository 의 책임

| 책임 | 예 |
| --- | --- |
| Aggregate 저장 / 조회 | `save(order)`, `findById(id)` |
| 도메인 의미가 있는 검색 | `findByCustomer`, `findPendingOrders` |
| 영속화 detail 차단 | SQL / JPA / Mongo 가 인터페이스에 없음 |
| 트랜잭션 무관 | 트랜잭션은 Service / Use Case 가 시작 |

→ Repository 에 `count` / `findAll` / 페이지네이션은 신중하게. 보통 read model 이 별도.

---

## 8. Domain Service

### 8.1 정의

한 Entity 또는 VO 에 자연스럽게 속하지 않는 도메인 로직을 담는 **stateless** 객체.

### 8.2 식별

| 신호 | 결과 |
| --- | --- |
| 로직이 여러 Entity 를 동시 참조 | Domain Service 후보 |
| 로직이 자기 자신을 호출하는 Entity 가 없음 | Domain Service 후보 |
| 로직이 외부 도메인 시스템 (환율, 신용 평가) 의존 | Domain Service 후보 |
| 로직이 stateless | Domain Service 가능 |

### 8.3 application service 와 차이

| 항목 | Domain Service | Application Service / Use Case |
| --- | --- | --- |
| 위치 | 도메인 계층 | 응용 계층 |
| 책임 | 도메인 규칙 | 흐름 조율 / 트랜잭션 / 외부 통신 |
| 트랜잭션 | 시작 안 함 | 시작 |
| 의존 | Domain 만 | Repository / Domain Service / 외부 |

### 8.4 예

```java
// Domain Service — 환율 변환 (한 Money 객체로 표현 불가)
interface CurrencyConverter {
    Money convert(Money source, Currency target);
}

// Domain Service — 신용 평가 (여러 도메인 객체 + 외부 의존)
interface CreditAssessment {
    CreditScore assess(Customer customer, List<Order> orderHistory);
}

// Application Service — 트랜잭션 / 흐름 조율
@Transactional
class PlaceOrder {
    private final OrderRepository orders;
    private final CreditAssessment credit;
    private final DomainEventPublisher events;

    OrderId handle(PlaceOrderCommand cmd, Customer customer) {
        CreditScore score = credit.assess(customer, orders.findByCustomer(customer.id()));
        if (!score.isAcceptable()) throw new CreditTooLowException();

        Order order = Order.create(customer.id(), cmd.items());
        orders.save(order);
        events.publish(new OrderPlaced(order.id()));
        return order.id();
    }
}
```

---

## 9. Domain Event

### 9.1 정의

도메인에서 발생한 의미 있는 사실. 보통 past tense.

### 9.2 구조

```java
record OrderPlaced(
    OrderId orderId,
    CustomerId customerId,
    Money totalPrice,
    Instant occurredAt
) implements DomainEvent { }
```

### 9.3 발행

| 시점 | 방법 |
| --- | --- |
| Aggregate 안에서 발생 | `order.place()` 가 event 를 자기 안에 누적 |
| 저장 전 | `order.uncommittedEvents()` 로 가져옴 |
| 저장 후 | 영속화 트랜잭션 안에서 Outbox 로 기록 → 별도 publisher |

```java
class Order {
    private final List<DomainEvent> uncommitted = new ArrayList<>();

    void place() {
        // ... 상태 변경 ...
        uncommitted.add(new OrderPlaced(id, customerId, totalPrice, Instant.now()));
    }

    List<DomainEvent> pullEvents() {
        List<DomainEvent> events = List.copyOf(uncommitted);
        uncommitted.clear();
        return events;
    }
}
```

### 9.4 효과

| 효과 | 결과 |
| --- | --- |
| Aggregate 간 결합 ↓ | 다른 Aggregate 는 event 만 수신 |
| eventual consistency | 트랜잭션 분리 → 가용성 ↑ |
| 외부 통합 | event → 다른 Bounded Context / 외부 시스템 |
| audit log | event 자체가 변경 이력 |

### 9.5 Outbox 패턴

DB 트랜잭션 안에 event 를 별도 outbox 테이블에 기록 → 비동기 publisher 가 메시지 큐로 발송. transaction + message 일관성 보장.

상세: [[../distributed-systems/distributed-systems]].

---

## 10. Factory

### 10.1 정의

복잡한 Aggregate 생성을 캡슐화하는 메커니즘. constructor 가 너무 복잡할 때 사용.

### 10.2 형태

| 형태 | 위치 | 예 |
| --- | --- | --- |
| Aggregate Root 의 static factory method | Root 자체 | `Order.create(customer, items)` |
| 별도 Factory class | 도메인 / 응용 계층 | `OrderFactory.placeFromCart(cart, customer)` |
| Builder | 인자 많을 때 | `Order.builder().customer(c).item(i).build()` |

### 10.3 예

```java
class OrderFactory {
    private final ProductRepository products;
    private final PricingService pricing;

    Order placeFromCart(Cart cart, Customer customer) {
        List<OrderItem> items = cart.items().stream()
            .map(ci -> {
                Product product = products.findById(ci.productId())
                    .orElseThrow(ProductNotFoundException::new);
                Money price = pricing.priceFor(product, customer);
                return new OrderItem(product.id(), price, ci.quantity());
            })
            .toList();
        return Order.create(customer.id(), items);
    }
}
```

→ 외부 의존이 필요한 생성은 별도 Factory. 단순 생성은 static factory method.

---

## 11. 7 pattern 의 관계 그래프

```
                       Bounded Context (strategic)
                              │
              ┌───────────────┼───────────────┐
              │               │               │
        [Aggregate]      [Aggregate]     [Aggregate]
              │               │               │
              ▼               ▼               ▼
        [Aggregate Root (Entity)]        ...
              │
              ├─ child Entity (e.g. OrderItem)
              ├─ VO (e.g. Money, Address)
              └─ uncommittedEvents → [Domain Event]
              ▲
              │ save / find
              │
        [Repository (interface)] ← 도메인 계층
                  ▲
                  │ implements
        [Repository (impl)] ← infrastructure 계층

[Application Service / Use Case]
   ├─ Repository 호출
   ├─ Aggregate method 호출
   ├─ Domain Service 호출
   └─ Domain Event 발행

[Domain Service] — stateless, 한 Aggregate 에 안 속하는 도메인 로직
[Factory] — 복잡한 Aggregate 생성
```

---

## 12. 백엔드 도메인 예 — 전체 구조

```java
// === Aggregate Root ===
class Order {
    private final OrderId id;
    private final CustomerId customerId;
    private final List<OrderItem> items;
    private OrderStatus status;
    private Money totalPrice;
    private final List<DomainEvent> uncommitted = new ArrayList<>();

    static Order create(CustomerId customerId, List<OrderItem> items) {
        if (items.isEmpty()) throw new EmptyOrderException();
        Order order = new Order(OrderId.generate(), customerId, items, OrderStatus.DRAFT);
        order.recalculateTotal();
        return order;
    }

    void place() {
        if (status != OrderStatus.DRAFT) throw new IllegalStateException();
        status = OrderStatus.PLACED;
        uncommitted.add(new OrderPlaced(id, customerId, totalPrice, Instant.now()));
    }

    void cancel() {
        if (status == OrderStatus.SHIPPED) throw new IllegalStateException();
        status = OrderStatus.CANCELLED;
        uncommitted.add(new OrderCancelled(id, Instant.now()));
    }

    List<DomainEvent> pullEvents() { /* */ }
}

// === Child Entity / VO ===
class OrderItem {
    private final ProductId productId;
    private final Money unitPrice;
    private final Quantity quantity;
    Money subtotal() { return unitPrice.multiply(quantity.value()); }
}

// === Repository (interface in domain layer) ===
interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
}

// === Domain Service ===
interface PricingService {
    Money priceFor(Product product, Customer customer);
}

// === Application Service ===
@Transactional
class PlaceOrder {
    private final OrderRepository orders;
    private final OrderFactory factory;
    private final DomainEventPublisher events;

    OrderId handle(PlaceOrderCommand cmd) {
        Order order = factory.placeFromCart(cmd.cart(), cmd.customer());
        order.place();
        orders.save(order);
        events.publishAll(order.pullEvents());
        return order.id();
    }
}
```

---

## 13. ORM 과의 결합

| 패턴 | 권장 |
| --- | --- |
| Active Record (Django Model, Rails ActiveRecord) | 단순 CRUD — DDD 의 Aggregate 추상이 약함 |
| Data Mapper (Hibernate 의 detached + Domain Model 분리) | DDD 와 자연스러운 결합 — Domain 이 ORM 무지 |
| JPA `@Embeddable` for VO | Money, Address 등 inline 저장 |
| JPA `@Entity` for Aggregate Root | id + cascade 설정 |
| event sourcing | Aggregate 의 state 대신 event sequence 저장 (별도 패턴) |

상세: [[oop-for-backend]] §9.

---

## 14. 흔한 실패 모드

| 실패 | 원인 | 정정 |
| --- | --- | --- |
| Aggregate 가 너무 큼 (Customer 가 Order 도 보유) | 경계 잘못 | Customer / Order 분리 + CustomerId 참조 |
| 두 Aggregate 동시 수정 (트랜잭션 안) | 일관성 경계 위반 | Domain Event + eventual consistency |
| `OrderItemRepository` 별도 존재 | Aggregate 단위 위반 | OrderRepository 만 |
| 모든 로직이 Service 에 | anemic Aggregate | 행위를 Aggregate Root 로 이동 |
| Domain Service 가 stateful | DI container 의 singleton 가정 깨짐 | stateless 보장 |
| Domain Event 가 발행 후 영속화 실패 | transaction + message 비일관 | Outbox 패턴 |
| Aggregate 가 외부 ORM annotation 의존 | Domain ↔ Infrastructure 결합 | 별도 ORM Entity + Mapper |
| Aggregate 내부 reference 가 다른 Aggregate 객체 | id 만 참조 규칙 위반 | id 로만 참조 |

---

## 15. 참고

- [[object-oriented-programming|↑ OOP hub]]
- [[concepts]]
- [[value-objects]]
- [[oop-for-backend]] §8
- [[../software-engineering/software-engineering]] §5.4 strategic DDD
- [[../distributed-systems/distributed-systems]] CQRS / Event Sourcing / Outbox
- Eric Evans — "Domain-Driven Design" (2003) — tactical pattern 원전
- Vaughn Vernon — "Implementing Domain-Driven Design" (2013) — Aggregate 4 rules
- Vaughn Vernon — "Effective Aggregate Design" (3 part article, 2011)
- Martin Fowler — "Patterns of Enterprise Application Architecture" (2002) — Repository / Domain Model 패턴
