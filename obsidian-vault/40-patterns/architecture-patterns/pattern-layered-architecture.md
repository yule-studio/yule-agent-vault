---
title: "pattern-layered-architecture — 표준 N-Tier (Controller / Service / Repository / Domain)"
kind: knowledge
project: patterns
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-18T23:00:00+09:00
tags: [patterns, architecture, layered, n-tier]
home_hub: architecture-patterns
related:
  - "[[architecture-patterns]]"
  - "[[pattern-hexagonal-architecture]]"
  - "[[pattern-clean-architecture]]"
  - "[[../../20-areas/computer-science/object-oriented-programming/oop-for-backend]]"
---

# pattern-layered-architecture — 표준 N-Tier

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-18 | engineering-agent/tech-lead | 최초 정리 |

**[[architecture-patterns|↑ architecture-patterns]]**

---

## 1. 의도

응용 서비스를 **수직 계층** 으로 분리해 각 계층이 명확한 책임을 갖게 한다. 가장 보편적인 백엔드 구조이며 Spring / Django / Rails / Laravel 의 기본 패턴.

원전: "Patterns of Enterprise Application Architecture" (Fowler 2002) — Service Layer + Domain Model + Data Mapper / Active Record.

---

## 2. 구조

```
HTTP request
   ↓
┌──────────────────────────────┐
│ Presentation                  │  ← Controller (HTTP / JSON)
└────────────┬─────────────────┘
             ↓
┌──────────────────────────────┐
│ Application (Service)         │  ← Use Case / Transaction 경계
└────────────┬─────────────────┘
             ↓
┌──────────────────────────────┐
│ Domain                        │  ← Entity / Value Object / Business Rules
└────────────┬─────────────────┘
             ↓
┌──────────────────────────────┐
│ Infrastructure (Repository)   │  ← DB / 외부 API / 메시지 큐
└──────────────────────────────┘
             ↓
            DB
```

| 계층 | 책임 |
| --- | --- |
| **Presentation** | HTTP / JSON 파싱 + DTO 변환 + 응답 코드 결정 |
| **Application (Service)** | 비즈니스 흐름 조율 + Transaction 경계 |
| **Domain** | 비즈니스 규칙 / invariant / 도메인 객체 |
| **Infrastructure** | DB / 외부 API / 메시지 큐 / 캐시 |

---

## 3. 의존 방향

```
Presentation  →  Application  →  Domain
                       │
                       ↓
                Infrastructure
```

→ 의존이 위에서 아래로. **strict layered** 는 인접 계층만 호출, **relaxed layered** 는 건너뛰기 허용.

| 변형 | 규칙 |
| --- | --- |
| strict | Controller → Service → Repository — Controller 가 Repository 직접 호출 불가 |
| relaxed | Controller 가 단순 조회는 Repository 직접 호출 OK |

→ 보통 strict 권장. Hexagonal / Clean 으로 발전 시 DIP 적용 (Domain 이 Repository interface 정의, Infrastructure 가 구현).

---

## 4. 적용 — 폴더 구조 예 (Java / Spring)

```
src/main/java/com/example/order/
├── controller/                      ← Presentation
│   ├── OrderController.java
│   └── dto/
│       ├── PlaceOrderRequest.java
│       └── OrderResponse.java
│
├── service/                         ← Application
│   ├── OrderService.java
│   └── exception/
│       └── OrderNotFoundException.java
│
├── domain/                          ← Domain
│   ├── Order.java
│   ├── OrderItem.java
│   ├── Money.java
│   └── OrderStatus.java
│
└── repository/                      ← Infrastructure
    ├── OrderRepository.java          (interface)
    └── JpaOrderRepository.java       (impl)
```

→ Hexagonal 의 `application/port/out/` 분리 없이 Repository interface 와 impl 이 같은 패키지. DIP 의 약한 적용.

---

## 5. 적용 — 코드 예

### 5.1 Controller

```java
@RestController
@RequestMapping("/orders")
class OrderController {
    private final OrderService orderService;

    OrderController(OrderService orderService) { this.orderService = orderService; }

    @PostMapping
    OrderResponse create(@Valid @RequestBody PlaceOrderRequest req) {
        Order order = orderService.placeOrder(req.toCommand());
        return OrderResponse.from(order);
    }

    @GetMapping("/{id}")
    OrderResponse get(@PathVariable String id) {
        return OrderResponse.from(orderService.findById(new OrderId(id)));
    }
}
```

### 5.2 Service

```java
@Service
@Transactional
class OrderService {
    private final OrderRepository orderRepository;
    private final CustomerRepository customerRepository;

    OrderService(OrderRepository orderRepository, CustomerRepository customerRepository) {
        this.orderRepository = orderRepository;
        this.customerRepository = customerRepository;
    }

    Order placeOrder(PlaceOrderCommand cmd) {
        Customer customer = customerRepository.findById(cmd.customerId())
            .orElseThrow(() -> new CustomerNotFoundException(cmd.customerId()));
        Order order = Order.create(customer.id(), cmd.items());
        return orderRepository.save(order);
    }

    @Transactional(readOnly = true)
    Order findById(OrderId id) {
        return orderRepository.findById(id)
            .orElseThrow(() -> new OrderNotFoundException(id));
    }
}
```

### 5.3 Domain

```java
class Order {
    private final OrderId id;
    private final CustomerId customerId;
    private final List<OrderItem> items;
    private OrderStatus status;
    private final Money totalPrice;

    static Order create(CustomerId customerId, List<OrderItem> items) {
        if (items.isEmpty()) throw new EmptyOrderException();
        return new Order(/* */);
    }

    void cancel() {
        if (status == OrderStatus.SHIPPED) {
            throw new IllegalStateException("cannot cancel shipped order");
        }
        status = OrderStatus.CANCELLED;
    }
}
```

### 5.4 Repository (Spring Data JPA)

```java
interface OrderRepository extends JpaRepository<Order, String> {
    Optional<Order> findByCustomerId(String customerId);
}
```

→ Spring Data JPA 가 interface 의 method name 으로 query 자동 생성. Repository 가 interface + JPA impl 자동 — Layered 의 단순화.

---

## 6. 핵심 규칙

| # | 규칙 |
| --- | --- |
| 1 | Controller 는 비즈니스 로직 0 — DTO 변환 + Service 호출만 |
| 2 | Service 가 `@Transactional` 의 경계 |
| 3 | Domain 객체가 비즈니스 규칙 보유 (anemic 회피) |
| 4 | Repository 는 영속화만 — 비즈니스 로직 X |
| 5 | DTO 와 Domain 객체 분리 — Controller 에서 변환 |
| 6 | 의존성 주입은 constructor |

---

## 7. Hexagonal / Clean 과의 차이

| 항목 | Layered | Hexagonal | Clean |
| --- | --- | --- | --- |
| 계층 수 | 4 (controller / service / domain / repo) | 3 (domain / app / adapter) | 4 (entity / use case / adapter / framework) |
| Domain 의 framework 의존 | 가능 (JPA `@Entity` 흔함) | 금지 | 금지 |
| Repository interface 위치 | Repository 패키지 | Application port | Use Case |
| use case 표현 | Service method | Application Service | Interactor + Input/Output Boundary |
| 학습 곡선 | 낮음 | 보통 | 큼 |
| over-engineering 위험 | 없음 | 보통 | 큼 |

→ Layered 는 Hexagonal 의 단순화. **DIP 의 약한 적용** (Domain 이 framework 의존 가능, Repository interface 가 도메인 외 위치).

---

## 8. 비용 / 트레이드오프

| 비용 | 의미 |
| --- | --- |
| Domain ↔ ORM 결합 | JPA `@Entity` 가 Domain 객체 = framework lock-in |
| 인프라 교체 어려움 | JPA → Mongo 시 Domain 도 영향 |
| 큰 도메인에서 god service | Service 가 비대화하기 쉬움 |

| 이점 | 의미 |
| --- | --- |
| 빠른 출시 | boilerplate 적음 |
| 학습 곡선 낮음 | Spring / Django 의 표준 |
| 신입 onboarding 빠름 | 표준 패턴 |
| 단순 CRUD 에 적합 | 도메인 규칙 적으면 충분 |

---

## 9. 적용 조건

| 조건 | 적합 |
| --- | --- |
| 단순 CRUD + 적은 도메인 규칙 | ★ 매우 적합 |
| MVP / startup 초기 | ★ 적합 |
| 팀 규모 작음 (2-5명) | ★ 적합 |
| Spring / Django / Rails 의 관례 따르기 | ★ 적합 |
| 도메인 매우 복잡 | △ Hexagonal / Clean 권장 |
| 인프라 교체 가능성 높음 | △ Hexagonal 권장 |
| 다중 클라이언트 (web / mobile / batch) | △ Hexagonal 권장 |
| 장기 유지보수 (5+ 년) | △ Hexagonal / Clean 권장 |

---

## 10. 흔한 실패 모드

| 실패 | 원인 | 정정 |
| --- | --- | --- |
| **Fat Controller** | Controller 가 비즈니스 로직 / SQL / 검증 보유 | Service 로 이동 |
| **God Service** | 한 Service 가 모든 주문 로직 + 검증 + 알림 + ... | 작은 Use Case 로 분리 |
| **Anemic Domain** | Domain 객체가 getter/setter 만 + Service 가 모든 비즈니스 | 도메인 method 로 행위 이동 ([[../../20-areas/computer-science/object-oriented-programming/anti-patterns]] §4.2) |
| **Repository 안 비즈니스 로직** | `OrderRepository.processOrder()` | Service 로 이동 |
| **Entity = ORM Entity** | `@Entity` 가 Domain — framework lock-in | Hexagonal 마이그레이션 검토 |
| **DTO 가 Domain 객체** | Request DTO 를 그대로 DB 저장 | DTO → Domain → ORM 분리 |
| **트랜잭션 누락** | 여러 Repository 호출이 트랜잭션 밖 | Service 의 `@Transactional` |
| **N+1 query** | Lazy loading 무지 | fetch join / DataLoader |
| **Service 가 다른 Service 호출 무한 chain** | use case 경계 모호 | Application 계층 분리 |

---

## 11. 마이그레이션 — Layered → Hexagonal

다음 신호가 누적되면 Hexagonal 로 마이그레이션 검토.

| 신호 | 의미 |
| --- | --- |
| Service 가 god class 화 (1000+ 줄) | Use Case 단위 분리 필요 |
| Domain 객체가 anemic | Hexagonal 의 도메인 중심으로 |
| 새 외부 의존 (메시지 큐 / 외부 API) 추가 시 cascade | port + adapter 격리 필요 |
| 단위 테스트가 DB 의존 | DIP 강화 |
| 인프라 교체 검토 (JPA → Mongo / Stripe → PayPal) | port + adapter 필수 |

마이그레이션 절차: [[pattern-hexagonal-architecture]] §10.

---

## 12. 관련

- [[architecture-patterns|↑ architecture-patterns]]
- [[pattern-hexagonal-architecture]]
- [[pattern-clean-architecture]]
- [[../../20-areas/computer-science/object-oriented-programming/oop-for-backend|↗ OOP for backend]]
- [[../../20-areas/computer-science/object-oriented-programming/anti-patterns|↗ OOP anti-patterns]]
- [[../../20-areas/computer-science/software-engineering/software-engineering|↗ software engineering]]
- Martin Fowler — "Patterns of Enterprise Application Architecture" (2002) §1, §9
- Spring Framework docs — Application Architecture
- Django Best Practices — fat models / thin views
