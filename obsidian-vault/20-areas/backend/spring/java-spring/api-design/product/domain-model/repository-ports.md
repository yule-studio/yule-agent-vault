---
title: "Repository Ports — 도메인 인터페이스"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:34:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - domain-model
  - port
---

# Repository Ports — 도메인 인터페이스

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[domain-model|↑ hub]]**

---

## 1. Ports

```java
public interface ProductRepository {
    Product save(Product p);
    Optional<Product> findById(ProductId id);
    List<Product> findActiveByCategory(CategoryId cid, Pageable page);
}

public interface OrderRepository {
    Order save(Order o);
    Optional<Order> findById(OrderId id);
    List<Order> findExpiredPending(Duration timeout, Instant now);
}

public interface PaymentRepository {
    Payment save(Payment p);
    Optional<Payment> findById(PaymentId id);
    Optional<Payment> findByOrderId(OrderId id);
    Optional<Payment> findByPgPaymentKey(String provider, String key);
    Optional<Payment> findByIdempotencyKey(String key);
}

public interface InventoryRepository {
    Inventory save(Inventory i);
    Optional<Inventory> findBySkuId(SkuId id);
    List<Inventory> findLow();
}

public interface DigitalDeliveryRepository {
    DigitalDelivery save(DigitalDelivery d);
    Optional<DigitalDelivery> findById(DeliveryId id);
    List<DigitalDelivery> findPending(int limit);    // worker
    Optional<DigitalDelivery> findByTokenHash(String hash);
}
```

---

## 2. 왜 도메인 layer 에 인터페이스

- 도메인 외부 의존성 0 (DB / Spring 알 필요 X).
- 테스트 시 in-memory 구현으로 fast.
- Adapter 선택 가능 (JPA / MyBatis / JDBC).

---

## 3. JPA 어댑터

```java
// infrastructure
@Component
@RequiredArgsConstructor
public class JpaProductRepository implements ProductRepository {
    private final ProductJpaRepository jpa;

    public Product save(Product p) { return jpa.save(ProductEntity.from(p)).toDomain(); }
    public Optional<Product> findById(ProductId id) { ... }
    public List<Product> findActiveByCategory(...) { ... }
}
```

자세히: [[../../database/jpa|↗ JPA recipe]].

---

## 4. 함정

### 함정 1 — Spring `@Repository` 가 도메인에 있음
도메인이 Spring 알게 됨.
→ 인터페이스만 도메인, 구현은 infrastructure.

### 함정 2 — JPA entity 가 도메인
JPA 의 변경 (lazy / proxy) 가 도메인에 새어 들어옴.
→ 별도 ProductEntity + toDomain() 매퍼.

### 함정 3 — query 메서드 폭증
Repository 가 30 메서드.
→ Specification 패턴 또는 CQRS 분리.

---

## 5. 관련

- [[domain-model|↑ hub]]
- [[../architecture]]
- [[../../database/jpa|↗ JPA recipe]]
- [[../../database/mybatis|↗ MyBatis recipe]]
