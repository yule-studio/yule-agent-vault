---
title: "OOP for backend — 프레임워크에서의 적용 (Spring / Django / Rails)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-18T19:30:00+09:00
tags: [computer-science, oop, backend, spring, django, rails]
home_hub: object-oriented-programming
related:
  - "[[object-oriented-programming]]"
  - "[[concepts]]"
  - "[[solid-principles]]"
  - "[[good-vs-bad-oop]]"
  - "[[anti-patterns]]"
  - "[[../software-engineering/software-engineering]]"
---

# OOP for backend — 프레임워크에서의 적용 (Spring / Django / Rails)

**[[object-oriented-programming|↑ object-oriented-programming]]**

---

## 1. 목적

본 문서는 백엔드 프레임워크 (Spring / Django / Rails / NestJS) 안에서 OOP 원칙을 적용하는 표준 패턴을 정의한다.

본 문서가 정의하는 것:
- 백엔드의 표준 계층 (Controller / Service / Repository / Domain / DTO) 의 책임 분리
- 각 계층에서 OOP 원칙 (SRP / DIP / 캡슐화 / tell don't ask) 의 적용
- 프레임워크별 특수성과 그 안에서의 좋은 OOP 패턴
- Entity vs Aggregate vs Value Object vs DTO 의 구분
- ORM 의 active record / data mapper 선택의 OOP 영향
- 백엔드에서 자주 발생하는 OOP 안티패턴

본 문서가 정의하지 않는 것:
- 각 프레임워크의 syntax 세부 — 프레임워크 docs
- DB 모델링 / 인덱스 — 별도
- 트랜잭션 isolation level — 별도

---

## 2. 범위

| 구분 | 포함 |
| --- | --- |
| 대상 프레임워크 | Spring Boot (Java/Kotlin), Django (Python), Rails (Ruby), NestJS (TypeScript) |
| 대상 패턴 | 표준 계층 / DI / Repository / Domain Model |
| 제외 | API 설계 (REST / GraphQL / gRPC) — 별도 |
| 제외 | 분산 / 메시지 큐 / 이벤트 소싱 — [[../distributed-systems/distributed-systems]] |

---

## 3. 용어

| 용어 | 정의 |
| --- | --- |
| **Controller** | HTTP 요청을 수신하고 응답을 만드는 진입점. |
| **Service / Use Case** | 비즈니스 흐름을 조율하는 application 계층. |
| **Repository** | 영속 저장소 (DB / 외부 API) 접근을 추상화하는 계층. |
| **Domain Model** | 비즈니스 규칙과 invariant 를 가진 객체. |
| **Entity** | 식별자로 구분되는 도메인 객체. |
| **Value Object** | 식별자 없이 값으로 구분되는 immutable 도메인 객체. |
| **Aggregate** | 일관성 경계를 공유하는 Entity / Value Object 의 묶음. (DDD) |
| **DTO (Data Transfer Object)** | 계층 / 시스템 간 데이터 전달용 객체. 행위 없음. |
| **DI (Dependency Injection)** | 의존을 외부에서 주입하는 기법. constructor / setter / field. |
| **IoC container** | DI 를 자동화하는 framework 구성 요소. Spring / Guice / Dagger. |

---

## 4. 표준 계층 책임 분리

### 4.1 4 계층 모델

```
HTTP request
   ↓
[Controller]            ← HTTP / JSON ↔ 도메인 변환만
   ↓
[Service / UseCase]     ← 비즈니스 흐름 조율 (transaction 경계)
   ↓
[Domain Model]          ← invariant + 핵심 비즈니스 규칙
   ↓
[Repository (interface)] ← 영속화 abstraction
   ↓
[Repository (impl)]     ← 실제 ORM / SQL / 외부 API
   ↓
DB / external
```

### 4.2 각 계층의 책임 / 금지

| 계층 | 책임 | 금지 |
| --- | --- | --- |
| Controller | HTTP 파싱 + DTO 변환 + 응답 코드 결정 | 비즈니스 로직, SQL, transaction 관리 |
| Service | 여러 도메인 객체 협력 조율, transaction 경계 | HTTP / JSON 의존, view 형식 결정 |
| Domain Model | invariant 검증 + 도메인 규칙 | DB / HTTP / 프레임워크 의존 |
| Repository (interface) | 도메인 객체의 영속화 명세 | 구현 (interface 만) |
| Repository (impl) | 실제 영속화 | 비즈니스 로직 |

→ 핵심 규칙: **의존 방향은 안쪽 (Domain Model) 으로** — Domain 이 외부 (HTTP / DB / 프레임워크) 를 모름.

### 4.3 호출 흐름 예 (주문 생성)

```
POST /orders
   ↓
@RestController OrderController.createOrder(CreateOrderRequest req)
   ↓ DTO → command
@Service PlaceOrder.handle(PlaceOrderCommand cmd)
   ↓
domain: Order.create(customer, items)   ← invariant 검증
   ↓
OrderRepository.save(order)             ← interface
   ↓ (JpaOrderRepository 구현)
DB INSERT
   ↓
OrderController return OrderResponse (DTO)
```

---

## 5. Spring Boot 적용 예

### 5.1 Controller

```java
@RestController
@RequestMapping("/orders")
class OrderController {
    private final PlaceOrder placeOrder;
    private final OrderQueries queries;

    OrderController(PlaceOrder placeOrder, OrderQueries queries) {
        this.placeOrder = placeOrder;
        this.queries = queries;
    }

    @PostMapping
    OrderResponse create(@Valid @RequestBody CreateOrderRequest req) {
        OrderId id = placeOrder.handle(req.toCommand());
        return OrderResponse.from(queries.byId(id));
    }
}
```

| 적용 OOP 원칙 | 결과 |
| --- | --- |
| SRP | HTTP / JSON 변환만 |
| DIP | PlaceOrder / OrderQueries interface 의존 |
| ISP | 쿼리와 명령 분리 (CQS) |

### 5.2 Service (Use Case)

```java
@Service
@Transactional
class PlaceOrder {
    private final CustomerRepository customers;
    private final OrderRepository orders;
    private final DomainEventPublisher events;

    PlaceOrder(CustomerRepository customers, OrderRepository orders, DomainEventPublisher events) {
        this.customers = customers;
        this.orders = orders;
        this.events = events;
    }

    OrderId handle(PlaceOrderCommand cmd) {
        Customer customer = customers.find(cmd.customerId())
            .orElseThrow(() -> new CustomerNotFound(cmd.customerId()));
        Order order = Order.create(customer, cmd.items());
        orders.save(order);
        events.publish(new OrderPlaced(order.id()));
        return order.id();
    }
}
```

| 적용 OOP 원칙 | 결과 |
| --- | --- |
| SRP | 주문 생성 흐름 조율만 |
| DIP | 모든 의존이 interface |
| transaction | service 가 경계 |

### 5.3 Domain Model

```java
class Order {
    private final OrderId id;
    private final CustomerId customerId;
    private final List<OrderItem> items;
    private OrderStatus status;
    private final Money totalPrice;

    private Order(OrderId id, CustomerId customerId, List<OrderItem> items, Money totalPrice) {
        this.id = id;
        this.customerId = customerId;
        this.items = List.copyOf(items);
        this.status = OrderStatus.CREATED;
        this.totalPrice = totalPrice;
    }

    static Order create(Customer customer, List<OrderItem> items) {
        if (items.isEmpty()) throw new EmptyOrderException();
        Money total = items.stream()
            .map(OrderItem::subtotal)
            .reduce(Money.ZERO, Money::plus);
        Money discounted = customer.applyDiscount(total);
        return new Order(OrderId.generate(), customer.id(), items, discounted);
    }

    void cancel() {
        if (status == OrderStatus.SHIPPED) {
            throw new IllegalStateException("cannot cancel shipped order");
        }
        status = OrderStatus.CANCELLED;
    }

    OrderId id()           { return id; }
    OrderStatus status()   { return status; }
    Money totalPrice()     { return totalPrice; }
}
```

| 적용 OOP 원칙 | 결과 |
| --- | --- |
| 캡슐화 | constructor private, factory method 가 invariant 강제 |
| tell, don't ask | `order.cancel()` (외부가 status 를 set 하지 않음) |
| 프레임워크 무관 | `@Entity` 등 framework annotation 없음 |

### 5.4 Repository

```java
// interface — 도메인 계층이 정의
interface OrderRepository {
    void save(Order order);
    Optional<Order> find(OrderId id);
}

// impl — infrastructure 계층
@Repository
class JpaOrderRepository implements OrderRepository {
    private final OrderJpaEntityRepository jpaRepo;

    public void save(Order order) {
        OrderJpaEntity entity = OrderJpaEntity.from(order);
        jpaRepo.save(entity);
    }
    public Optional<Order> find(OrderId id) {
        return jpaRepo.findById(id.value()).map(OrderJpaEntity::toDomain);
    }
}

// JPA entity — 도메인과 분리 (data mapper 패턴)
@Entity
@Table(name = "orders")
class OrderJpaEntity {
    @Id private String id;
    @Column private String customerId;
    @Column private String status;
    // ... ORM 전용
}
```

| 적용 OOP 원칙 | 결과 |
| --- | --- |
| DIP | 도메인 → interface, impl 은 infrastructure |
| 캡슐화 | 도메인 객체가 ORM 의존 없음 |
| 교체 가능 | JPA → JDBC → Mongo 교체 시 도메인 영향 0 |

---

## 6. Django 적용 예

### 6.1 Django 의 active record 패턴

Django ORM 의 Model 은 active record (행위 + 데이터 + DB 접근이 한 클래스).

```python
# Django default — Model 이 DB 접근
class Order(models.Model):
    customer = models.ForeignKey(Customer, on_delete=models.CASCADE)
    status = models.CharField(max_length=20, default="CREATED")
    total_price = models.DecimalField(max_digits=10, decimal_places=2)

    def cancel(self):
        if self.status == "SHIPPED":
            raise ValueError("cannot cancel shipped order")
        self.status = "CANCELLED"
        self.save()    # ← DB 접근이 도메인 method 안에
```

| 평가 | 결과 |
| --- | --- |
| 단순 CRUD | 빠름, 적합 |
| 복잡한 도메인 | DB / 비즈니스 결합 — 테스트 어려움 |

### 6.2 Django + 별도 도메인 계층 (Hexagonal)

```python
# domain/order.py — Django 무관
@dataclass
class Order:
    id: OrderId
    customer_id: CustomerId
    items: tuple[OrderItem, ...]
    status: OrderStatus
    total_price: Money

    @classmethod
    def create(cls, customer: Customer, items: list[OrderItem]) -> "Order":
        if not items:
            raise EmptyOrderError
        total = sum((i.subtotal() for i in items), Money.ZERO)
        return cls(
            id=OrderId.generate(),
            customer_id=customer.id,
            items=tuple(items),
            status=OrderStatus.CREATED,
            total_price=customer.apply_discount(total),
        )

    def cancel(self) -> None:
        if self.status == OrderStatus.SHIPPED:
            raise CannotCancelShippedError
        self.status = OrderStatus.CANCELLED


# repositories/order.py — interface
class OrderRepository(Protocol):
    def save(self, order: Order) -> None: ...
    def find(self, id: OrderId) -> Order | None: ...


# infrastructure/django_order_repository.py
class DjangoOrderRepository:
    def save(self, order: Order) -> None:
        OrderModel.objects.update_or_create(
            id=order.id.value,
            defaults={
                "customer_id": order.customer_id.value,
                "status": order.status.value,
                "total_price": order.total_price.amount,
            }
        )
```

| 평가 | 결과 |
| --- | --- |
| 도메인 테스트 | DB 없이 가능 |
| Django 무관 | Flask / FastAPI 로 마이그레이션 가능 |
| 비용 | 두 계층 모델링 + 변환 코드 |

→ 단순 CRUD 는 active record, 복잡 도메인은 분리 계층.

---

## 7. Rails 적용 예

### 7.1 Rails 의 fat model 전통

Rails 는 "fat models, skinny controllers" 를 권장. 비즈니스 로직을 Model 에 둠.

```ruby
class Order < ApplicationRecord
    belongs_to :customer
    has_many :items

    validates :items, presence: true
    validate :ensure_not_shipped, on: :cancel

    def cancel!
        raise CannotCancelError if status == "shipped"
        update!(status: "cancelled")
    end

    def total_price
        items.sum { |i| i.price * i.quantity } * customer.discount_rate
    end
end
```

| 평가 | 결과 |
| --- | --- |
| 빠른 개발 | OK |
| Model 비대화 | Concern / Service Object / Form Object 로 분리 |

### 7.2 Rails 의 Service Object 패턴

복잡한 비즈니스 흐름은 Service Object 로 분리.

```ruby
# app/services/place_order.rb
class PlaceOrder
    def initialize(customer_id:, items:)
        @customer_id = customer_id
        @items = items
    end

    def call
        ActiveRecord::Base.transaction do
            customer = Customer.find(@customer_id)
            order = Order.create!(customer: customer, items: @items)
            DomainEventPublisher.publish(OrderPlaced.new(order.id))
            order
        end
    end
end

# controller
def create
    order = PlaceOrder.new(customer_id: params[:customer_id], items: params[:items]).call
    render json: OrderSerializer.new(order)
end
```

---

## 8. Entity / Value Object / Aggregate / DTO 구분

### 8.1 Entity

식별자로 구분. 같은 값이어도 식별자가 다르면 다른 객체.

```java
class Order {                  // Entity
    private final OrderId id;  // identity
    // ...
    @Override public boolean equals(Object o) {
        return o instanceof Order other && id.equals(other.id);
    }
}
```

### 8.2 Value Object

식별자 없음. 값으로 구분. **immutable**.

```java
record Money(BigDecimal amount, Currency currency) {
    Money {
        if (currency == null) throw new IllegalArgumentException();
        amount = amount.setScale(currency.getDefaultFractionDigits());
    }
    Money plus(Money other) {
        if (!currency.equals(other.currency)) throw new CurrencyMismatchException();
        return new Money(amount.add(other.amount), currency);
    }
}
```

### 8.3 Aggregate

일관성 경계를 공유하는 Entity / Value Object 의 묶음. 외부에서는 **Aggregate Root** 만 참조.

```
Aggregate: Order
├── OrderId (identity)
├── List<OrderItem> (value objects)
└── Money totalPrice (value object)

외부는 Order 만 참조, OrderItem 은 Order 통해서만 접근.
```

### 8.4 DTO

계층 간 / 시스템 간 데이터 전달. 행위 없음. immutable 권장.

```java
record CreateOrderRequest(
    String customerId,
    List<OrderItemDto> items
) {
    PlaceOrderCommand toCommand() {
        return new PlaceOrderCommand(
            new CustomerId(customerId),
            items.stream().map(OrderItemDto::toDomain).toList()
        );
    }
}
```

→ DTO 와 Domain Model 은 분리. 같은 데이터를 가지더라도 다른 객체.

---

## 9. ORM 의 패턴 선택

### 9.1 Active Record (Rails / Django)

`Order` 가 자기 DB 접근 method 보유 (`order.save()`).

| 장점 | 단점 |
| --- | --- |
| 적은 코드 | 도메인과 영속 결합 |
| 빠른 prototype | 복잡 도메인에서 테스트 / 마이그레이션 어려움 |

### 9.2 Data Mapper (Hibernate / SQLAlchemy classic)

별도 Mapper 가 객체 ↔ DB 변환. Domain Model 은 영속 무관.

| 장점 | 단점 |
| --- | --- |
| 도메인 / 영속 분리 | 더 많은 코드 |
| 복잡 도메인에 적합 | 학습 곡선 |

### 9.3 결정 기준

| 조건 | 권장 |
| --- | --- |
| 단순 CRUD, 도메인 규칙 적음 | Active Record |
| 복잡한 비즈니스 규칙, 테스트 중요 | Data Mapper |
| 다중 DB / 마이그레이션 가능성 | Data Mapper |
| 작은 팀, 빠른 출시 | Active Record |

---

## 10. DI / IoC 의 적용

### 10.1 DI 의 3 가지 방법

| 방법 | 예 | 평가 |
| --- | --- | --- |
| **Constructor 주입** | `OrderService(OrderRepository repo)` | 권장 (immutable, 명시적) |
| **Setter 주입** | `setOrderRepository(...)` | optional dependency 만 |
| **Field 주입** | `@Autowired field` | 비권장 (테스트 어려움, hidden) |

### 10.2 IoC container 사용

```java
// Spring
@Service
class OrderService {
    private final OrderRepository repo;

    OrderService(OrderRepository repo) {     // ← 자동 주입
        this.repo = repo;
    }
}
```

→ 어떤 구현이 주입될지는 @Bean 설정으로 제어. 테스트 시 mock 주입 가능.

### 10.3 DI 의 한계

| 한계 | 보완 |
| --- | --- |
| 자동 wiring 의 magic | constructor 주입으로 명시 |
| circular dependency | 책임 재분배, lazy 주입 |
| 너무 많은 의존 (생성자 인자 7+) | SRP 위반 — 책임 재분배 |

---

## 11. 백엔드 특유의 OOP 안티패턴

| 안티패턴 | 신호 |
| --- | --- |
| **Fat Controller** | Controller 가 비즈니스 로직 / SQL / 검증 모두 | Service 로 이동 |
| **Service Layer 폭주** | 모든 로직이 Service 에 — Domain 은 anemic | Domain Model 에 행위 이동 |
| **Entity = ORM Entity** | JPA `@Entity` 가 도메인 = anemic + framework lock-in | Domain Model + ORM Entity 분리 |
| **Repository 안 비즈니스 로직** | `OrderRepository.processOrder()` 같은 method | Use case / Service 로 이동 |
| **God Service** | `OrderService` 가 모든 주문 로직 보유 | 작은 Use Case 로 분리 |
| **DTO 가 도메인 객체** | request DTO 를 그대로 DB 저장 | DTO → Domain → ORM 분리 |
| **Magic Annotation 폭주** | `@Cacheable`, `@Async`, `@Retryable` 가 너무 많음 | 명시적 호출 / wrapper |
| **트랜잭션 누락** | 여러 repository 호출이 트랜잭션 밖 | Service 의 `@Transactional` |
| **N+1 query** | Lazy loading 의 무지 | fetch join / DataLoader |

---

## 12. 프레임워크 별 권장 패턴 요약

| 프레임워크 | 단순 CRUD | 복잡 도메인 |
| --- | --- | --- |
| Spring Boot | `@RestController` + `@Service` + JPA Entity | Controller + Use Case + Domain + Repo interface + JPA impl |
| Django | Model + ModelViewSet + serializer | View + Service + Domain dataclass + Repo (Protocol) + Django ORM impl |
| Rails | Fat Model + Controller | Model + Service Object + Form Object + Repository (gem) |
| NestJS | `@Controller` + Service + TypeORM Entity | Controller + Use Case + Domain class + Repo interface + TypeORM impl |

---

## 13. 적용 체크 항목

새 백엔드 모듈 작성 시 다음 확인.

| 항목 | 점검 |
| --- | --- |
| Controller 가 비즈니스 로직 0 인가 |  |
| Service 가 transaction 경계인가 |  |
| Domain Model 이 프레임워크 annotation 의존 0 인가 |  |
| Repository 가 interface 로 의존되는가 |  |
| invariant 검증이 Domain Model 의 constructor / method 안에 있는가 |  |
| getter/setter 만 있는 anemic 객체가 없는가 |  |
| DTO 와 Domain Model 이 분리되는가 |  |
| 의존성이 모두 constructor 주입인가 |  |
| 단위 테스트가 외부 (DB / HTTP) 의존 없이 가능한가 |  |

---

## 14. 참고

- [[object-oriented-programming|↑ OOP hub]]
- [[concepts]]
- [[solid-principles]]
- [[good-vs-bad-oop]]
- [[anti-patterns]]
- [[../software-engineering/software-engineering]] §5 아키텍처 패턴
- [[../design-patterns/design-patterns]]
- [[../../backend|↗ backend area]]
- Eric Evans — "Domain-Driven Design" (2003)
- Vaughn Vernon — "Implementing Domain-Driven Design" (2013)
- Robert C. Martin — "Clean Architecture" (2017) — 4 계층의 의존 방향
- Martin Fowler — "Patterns of Enterprise Application Architecture" (2002) — Active Record / Data Mapper / Service Layer
