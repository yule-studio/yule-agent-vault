---
title: "pattern-hexagonal-architecture — Ports & Adapters"
kind: knowledge
project: patterns
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-18T22:00:00+09:00
tags: [patterns, architecture, hexagonal, ports-and-adapters]
home_hub: architecture-patterns
related:
  - "[[architecture-patterns]]"
  - "[[pattern-clean-architecture]]"
  - "[[pattern-layered-architecture]]"
  - "[[../../20-areas/computer-science/object-oriented-programming/oop-for-backend]]"
  - "[[../../20-areas/computer-science/object-oriented-programming/ddd-tactical-patterns]]"
---

# pattern-hexagonal-architecture — Ports & Adapters

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-18 | engineering-agent/tech-lead | 최초 정리 |

**[[architecture-patterns|↑ architecture-patterns]]**

---

## 1. 의도

도메인 비즈니스 규칙을 외부 (UI / DB / 메시지 큐 / 외부 API) 의 변경으로부터 격리한다. 외부 ↔ 도메인의 모든 접점을 **port (interface)** + **adapter (구현)** 로 표현한다.

원전: Alistair Cockburn — "Hexagonal Architecture" (2005).

---

## 2. 구조

```
                ┌──────────────────────────┐
                │  Driving Adapter (HTTP)  │
                └────────────┬─────────────┘
                             │ (driving port)
        ┌────────────────────▼─────────────────────────┐
        │  Driving Adapter (CLI)                        │
        │  Driving Adapter (Scheduler)                  │
        │  Driving Adapter (Test)                       │
        └────────────────────┬─────────────────────────┘
                             │
              ┌──────────────▼──────────────────┐
              │   Application Service           │  ← Use Case
              │       │                         │
              │       ▼                         │
              │   ┌────────────────────┐        │
              │   │   Domain Model     │        │  ← Entity / VO / Aggregate
              │   └────────────────────┘        │
              │       │                         │
              │       ▼                         │
              │   Driven Port (interface)       │  ← OrderRepository, EmailSender ...
              └──────────────┬──────────────────┘
                             │
        ┌────────────────────▼─────────────────────────┐
        │  Driven Adapter (JPA)                         │
        │  Driven Adapter (SMTP)                        │
        │  Driven Adapter (Kafka)                       │
        │  Driven Adapter (S3)                          │
        └───────────────────────────────────────────────┘
```

| 부분 | 책임 |
| --- | --- |
| **Driving Adapter** | 외부 → app 으로의 진입 (HTTP / CLI / 스케줄러 / 테스트) |
| **Driving Port** | application 이 노출하는 use case 인터페이스 |
| **Application Service** | 도메인 흐름 조율 + 트랜잭션 |
| **Domain Model** | 비즈니스 규칙 / invariant |
| **Driven Port** | application 이 의존하는 외부 추상 (interface) |
| **Driven Adapter** | 외부와의 통신 구현 (JPA / Redis / SMTP / Kafka) |

→ "Hexagonal" 은 비유 — 6 각형의 각 변에 다른 adapter 가 붙을 수 있다는 의미. 실제 변 수는 무관.

---

## 3. 의존 방향

```
Driving Adapter ──→ Driving Port ──→ Application Service ──→ Domain Model
                                              │
                                              └──→ Driven Port (interface)
                                                          ▲
                                                          │ implements
                                              Driven Adapter
```

→ **모든 의존이 도메인 (안쪽) 으로 향한다.** Driven Adapter 는 Driven Port (인터페이스) 를 구현하므로 의존이 바깥쪽이 아닌 인터페이스 방향.

---

## 4. 적용 — 폴더 구조 예 (Java / Spring)

```
src/main/java/com/example/order/
├── domain/                          ← 도메인 — 외부 의존 0
│   ├── Order.java                   (Aggregate Root)
│   ├── OrderId.java                 (VO)
│   ├── OrderStatus.java
│   ├── Money.java                   (VO)
│   └── DomainEvent.java
│
├── application/                     ← Use Case + Port 정의
│   ├── port/
│   │   ├── in/                      (driving port)
│   │   │   └── PlaceOrderUseCase.java
│   │   └── out/                     (driven port)
│   │       ├── OrderRepository.java
│   │       ├── PaymentGateway.java
│   │       └── EmailSender.java
│   └── service/
│       └── PlaceOrderService.java   (implements PlaceOrderUseCase)
│
└── adapter/                         ← 외부 — 도메인 알지만 도메인은 모름
    ├── in/                          (driving adapter)
    │   ├── web/
    │   │   ├── OrderController.java
    │   │   └── PlaceOrderRequest.java (DTO)
    │   └── cli/
    │       └── PlaceOrderCommand.java
    └── out/                         (driven adapter)
        ├── persistence/
        │   ├── JpaOrderRepository.java (implements OrderRepository)
        │   └── OrderJpaEntity.java
        ├── payment/
        │   └── StripePaymentGateway.java (implements PaymentGateway)
        └── email/
            └── SmtpEmailSender.java (implements EmailSender)
```

| 폴더 | 의존 가능 대상 |
| --- | --- |
| `domain/` | (없음) — 자기 자신만 |
| `application/` | `domain/` |
| `adapter/in/` | `application/port/in/` (driving port) + DTO 변환 |
| `adapter/out/` | `application/port/out/` (driven port) + `domain/` (모델 변환) |

---

## 5. 적용 — 코드 예

### 5.1 Driving Port (use case interface)

```java
package com.example.order.application.port.in;

public interface PlaceOrderUseCase {
    OrderId place(PlaceOrderCommand command);
}
```

### 5.2 Driven Port (외부 의존 interface)

```java
package com.example.order.application.port.out;

public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
}

public interface PaymentGateway {
    PaymentResult charge(Money amount, PaymentMethod method);
}
```

### 5.3 Application Service

```java
package com.example.order.application.service;

@Service
@Transactional
class PlaceOrderService implements PlaceOrderUseCase {
    private final OrderRepository orders;
    private final PaymentGateway payments;
    private final DomainEventPublisher events;

    public OrderId place(PlaceOrderCommand cmd) {
        Order order = Order.create(cmd.customerId(), cmd.items());
        PaymentResult result = payments.charge(order.totalPrice(), cmd.paymentMethod());
        if (!result.isSuccess()) throw new PaymentFailedException(result);
        order.confirmPayment();
        orders.save(order);
        events.publishAll(order.pullEvents());
        return order.id();
    }
}
```

### 5.4 Driving Adapter (HTTP)

```java
package com.example.order.adapter.in.web;

@RestController
@RequestMapping("/orders")
class OrderController {
    private final PlaceOrderUseCase placeOrder;

    @PostMapping
    OrderResponse create(@Valid @RequestBody PlaceOrderRequest req) {
        OrderId id = placeOrder.place(req.toCommand());
        return new OrderResponse(id.value());
    }
}
```

### 5.5 Driven Adapter (JPA)

```java
package com.example.order.adapter.out.persistence;

@Repository
class JpaOrderRepository implements OrderRepository {
    private final OrderJpaEntityRepository jpa;

    public void save(Order order) {
        jpa.save(OrderJpaEntity.from(order));
    }

    public Optional<Order> findById(OrderId id) {
        return jpa.findById(id.value()).map(OrderJpaEntity::toDomain);
    }
}
```

→ Domain `Order` 와 `OrderJpaEntity` 는 분리. mapper 가 변환.

---

## 6. 핵심 규칙

| # | 규칙 | 검증 방법 |
| --- | --- | --- |
| 1 | `domain/` 은 외부 import 0 (Spring / JPA / Jackson 모두 X) | ArchUnit / dependency-cruiser |
| 2 | `application/` 은 `adapter/` import 안 함 | 모든 driven adapter 의존은 port 통해 |
| 3 | 도메인 객체와 ORM Entity / DTO 는 분리 | 두 class 가 다른 패키지 |
| 4 | Use case interface 는 application 이 정의 | adapter 가 호출만 |
| 5 | 모든 의존성 주입은 constructor | Spring `@Autowired` 도 생성자 |

---

## 7. 비용 / 트레이드오프

| 비용 | 의미 |
| --- | --- |
| 추가 코드량 | port interface + adapter mapper. 단순 CRUD 의 2-3 배 |
| 학습 곡선 | 패키지 / interface 구조 익숙해질 때까지 1-2 주 |
| 작은 변경의 PR 크기 | 한 endpoint 추가 = controller + DTO + use case + port + adapter |
| 빠른 prototype 부적합 | MVP 단계에선 over-engineering |

| 이점 | 의미 |
| --- | --- |
| 인프라 교체 자유 | JPA → Mongo / Stripe → PayPal 교체 시 도메인 영향 0 |
| 단위 테스트 작성 쉬움 | 도메인이 외부 의존 0 — Mock 없이도 가능 |
| 다중 진입점 | HTTP / CLI / 스케줄러 / 테스트 같은 use case 호출 |
| 도메인 보존 | framework migration 시 도메인 유지 |

---

## 8. 적용 조건

| 조건 | 적합 여부 |
| --- | --- |
| 도메인이 복잡 (invariant 다수) | ★ 매우 적합 |
| 외부 의존 다수 (DB / 메시지 큐 / 외부 API / 결제 / 인증) | ★ 매우 적합 |
| 다중 클라이언트 (web / mobile / batch) | ★ 적합 |
| 인프라 교체 / 마이그레이션 가능성 | ★ 적합 |
| 단순 CRUD (도메인 규칙 거의 없음) | ✗ over-engineering |
| MVP / prototype | ✗ 시간 부족 |

---

## 9. Clean Architecture / Layered 와의 차이

| 항목 | Hexagonal | Clean | Layered |
| --- | --- | --- | --- |
| 계층 수 | 2-3 (domain / app / adapter) | 4 (entity / use case / interface adapter / framework) | 3-4 (controller / service / repo / domain) |
| 의존 방향 | 안쪽 (도메인) 으로 | 안쪽 (entity) 으로 | 위에서 아래로 |
| 외부 격리 | port + adapter | boundary + use case interactor | repository interface |
| 학습 곡선 | 중 | 높음 | 낮음 |
| over-engineering 위험 | 보통 | 큼 | 없음 |

→ Hexagonal 은 Clean 의 단순화로 볼 수 있음. 4 계층 대신 도메인 / 응용 / 어댑터 3 영역.

---

## 10. 마이그레이션 — Layered → Hexagonal

| 단계 | 작업 |
| --- | --- |
| 1 | `domain/` 패키지 신설. 도메인 객체를 framework 무관하게 분리 |
| 2 | `application/port/out/` 에 Repository / EmailSender 등 interface 추출 |
| 3 | 기존 Repository / Service 를 `adapter/out/` 으로 이동 + interface 구현 |
| 4 | `application/service/` 에 Use Case 신설 (기존 Service 의 흐름 조율 부분) |
| 5 | Controller 는 `adapter/in/` 으로 이동 + Use Case 만 의존 |
| 6 | ArchUnit / dependency-cruiser 로 의존 방향 검증 |

→ 한 번에 전체 변환 X. 새 기능부터 hexagonal + 기존은 점진 마이그레이션.

---

## 11. 흔한 실패 모드

| 실패 | 원인 | 정정 |
| --- | --- | --- |
| Domain 이 `@Entity` / `@Component` 의존 | framework 결합 | 별도 ORM Entity + mapper |
| 모든 method 가 interface 로 추출됨 | over-engineering | 실제 교체 가능성 있는 곳만 |
| Application Service 가 너무 비대 | use case 분리 안 됨 | use case 별 service class |
| Adapter 가 다른 adapter 를 직접 import | 의존 방향 위반 | application port 통해 |
| Domain 객체가 외부 노출 (HTTP response) | DTO 분리 안 함 | adapter 가 DTO 변환 |
| ArchUnit / 검증 도구 없음 | 형식만 적용, 실제 의존 위반 | 자동 검증 도입 |
| 폴더 구조만 따르고 의존 검증 0 | 의도 안 지킴 | 빌드 / CI 에 의존 규칙 통합 |

---

## 12. 검증 — ArchUnit 예 (Java)

```java
@AnalyzeClasses(packages = "com.example.order")
class HexagonalArchitectureTest {

    @ArchTest
    static final ArchRule domain_must_not_depend_on_other_layers =
        noClasses().that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAnyPackage(
                "..application..",
                "..adapter..",
                "org.springframework..",
                "javax.persistence..",
                "jakarta.persistence.."
            );

    @ArchTest
    static final ArchRule application_must_not_depend_on_adapter =
        noClasses().that().resideInAPackage("..application..")
            .should().dependOnClassesThat().resideInAPackage("..adapter..");

    @ArchTest
    static final ArchRule controllers_only_depend_on_use_cases =
        classes().that().resideInAPackage("..adapter.in.web..")
            .should().onlyDependOnClassesThat()
            .resideInAnyPackage("..application.port.in..", "..adapter.in.web..", "java..", "javax..");
}
```

---

## 13. 관련

- [[architecture-patterns|↑ architecture-patterns]]
- [[pattern-clean-architecture]]
- [[pattern-layered-architecture]]
- [[../../20-areas/computer-science/object-oriented-programming/oop-for-backend|↗ OOP for backend]]
- [[../../20-areas/computer-science/object-oriented-programming/ddd-tactical-patterns|↗ DDD tactical patterns]]
- [[../../20-areas/computer-science/object-oriented-programming/solid-principles|↗ SOLID]]
- Alistair Cockburn — "Hexagonal Architecture" (2005)
- Tom Hombergs — "Get Your Hands Dirty on Clean Architecture" (2019) — 실용 가이드
- Vaughn Vernon — "Implementing Domain-Driven Design" §4 — Hexagonal + DDD
- ArchUnit — https://www.archunit.org
