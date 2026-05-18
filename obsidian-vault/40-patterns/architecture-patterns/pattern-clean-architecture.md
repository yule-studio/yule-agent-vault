---
title: "pattern-clean-architecture — Uncle Bob 4 동심원"
kind: knowledge
project: patterns
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-18T22:30:00+09:00
tags: [patterns, architecture, clean-architecture, uncle-bob]
home_hub: architecture-patterns
related:
  - "[[architecture-patterns]]"
  - "[[pattern-hexagonal-architecture]]"
  - "[[pattern-layered-architecture]]"
  - "[[../../20-areas/computer-science/object-oriented-programming/oop-for-backend]]"
---

# pattern-clean-architecture — Uncle Bob 4 동심원

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-18 | engineering-agent/tech-lead | 최초 정리 |

**[[architecture-patterns|↑ architecture-patterns]]**

---

## 1. 의도

Robert C. Martin 이 Hexagonal / Onion / DCI / BCE 등의 공통 패턴을 통합해 정의한 **4 동심원 + dependency rule**. 도메인 (Entity) 을 가장 안쪽에 두고, 모든 의존이 안쪽으로만 향하게 한다.

원전: Robert C. Martin — "Clean Architecture" (2017).

---

## 2. 4 동심원

```
                ┌────────────────────────────────────────────┐
                │           Frameworks & Drivers              │
                │   (Spring / Express / Django / DB / Web)    │
                │   ┌────────────────────────────────────┐   │
                │   │      Interface Adapters             │   │
                │   │   (Controller / Presenter / DTO /   │   │
                │   │    Gateway / Repository Impl)       │   │
                │   │   ┌──────────────────────────┐     │   │
                │   │   │    Application Use Cases  │     │   │
                │   │   │    (Interactor + Boundary)│     │   │
                │   │   │   ┌──────────────────┐    │     │   │
                │   │   │   │     Entities      │    │     │   │
                │   │   │   │  (Domain Model)   │    │     │   │
                │   │   │   └──────────────────┘    │     │   │
                │   │   └──────────────────────────┘     │   │
                │   └────────────────────────────────────┘   │
                └────────────────────────────────────────────┘

                의존 방향: 바깥쪽 → 안쪽 (단방향)
```

| 계층 | 책임 |
| --- | --- |
| 1. **Entities** | 기업 전체에서 재사용 가능한 비즈니스 규칙. 가장 일반적이고 변경 가능성 낮음. |
| 2. **Use Cases (Application)** | 애플리케이션 별 비즈니스 흐름. 한 use case = 한 시나리오. |
| 3. **Interface Adapters** | 외부 (web / DB / 외부 API) ↔ Use Case / Entity 간 변환. |
| 4. **Frameworks & Drivers** | 외부 도구 — Spring / Django / DB / 메시지 큐. |

---

## 3. Dependency Rule

> "Source code dependencies can only point inwards." — Uncle Bob

```
Frameworks  →  Adapters  →  Use Cases  →  Entities
                                              ▲ 모든 의존이 여기로
```

| 원칙 | 의미 |
| --- | --- |
| 안쪽 layer 는 바깥쪽 layer 를 모름 | Entity 는 Use Case 모름. Use Case 는 Adapter 모름. |
| 바깥쪽 layer 는 안쪽 layer 에 의존 | Adapter 는 Use Case interface 호출 |
| 안쪽이 바깥을 호출해야 할 때 → DIP | 안쪽이 interface 선언, 바깥이 구현 |

---

## 4. Use Case 의 구조 (Interactor + Boundary)

```
[Controller]
    │ Input Boundary (interface)
    ▼
[Use Case Interactor]                    ← Use Case 로직
    │
    │ Output Boundary (interface)
    ▼
[Presenter]                              ← Response 변환
```

| 컴포넌트 | 책임 |
| --- | --- |
| Input Boundary | Use Case 의 진입 interface (Controller 가 호출) |
| Use Case Interactor | 비즈니스 흐름 + Entity 호출 + Repository 호출 |
| Output Boundary | Use Case 의 응답 interface (Presenter 가 구현) |
| Presenter | response model 로 변환 (Controller 가 view 로 변환) |

→ Hexagonal 의 driving port / driven port 와 같은 개념. Clean 은 OutputBoundary 까지 명시.

---

## 5. 적용 — 폴더 구조 예 (Java)

```
src/main/java/com/example/order/
├── entities/                            ← layer 1 — 기업 비즈니스 규칙
│   ├── Order.java
│   ├── OrderId.java
│   ├── Money.java
│   └── OrderStatus.java
│
├── usecases/                            ← layer 2 — 애플리케이션 비즈니스 규칙
│   ├── placeorder/
│   │   ├── PlaceOrderInputBoundary.java    (interface)
│   │   ├── PlaceOrderRequestModel.java     (DTO)
│   │   ├── PlaceOrderInteractor.java       (implements InputBoundary)
│   │   ├── PlaceOrderOutputBoundary.java   (interface)
│   │   └── PlaceOrderResponseModel.java    (DTO)
│   └── port/
│       ├── OrderGateway.java               (Repository interface)
│       └── PaymentGateway.java
│
├── adapters/                            ← layer 3
│   ├── controllers/
│   │   └── OrderController.java        (input adapter)
│   ├── presenters/
│   │   └── PlaceOrderPresenter.java    (output adapter)
│   ├── gateways/
│   │   ├── JpaOrderGateway.java        (implements OrderGateway)
│   │   └── StripePaymentGateway.java   (implements PaymentGateway)
│   └── viewmodels/
│       └── OrderViewModel.java
│
└── frameworks/                          ← layer 4
    ├── web/                             (Spring Web 설정)
    ├── persistence/                     (JPA 설정 + Entity class)
    └── config/                          (Spring DI 설정)
```

---

## 6. 적용 — 코드 예

### 6.1 Entity

```java
package com.example.order.entities;

public class Order {                  // 외부 의존 0
    private final OrderId id;
    private final CustomerId customerId;
    private final List<OrderItem> items;
    private OrderStatus status;
    private Money totalPrice;

    public static Order create(CustomerId customerId, List<OrderItem> items) {
        if (items.isEmpty()) throw new EmptyOrderException();
        // ... invariant 검증 ...
    }

    public void place() { /* 도메인 규칙 */ }
    public void cancel() { /* */ }
}
```

### 6.2 Use Case

```java
package com.example.order.usecases.placeorder;

public interface PlaceOrderInputBoundary {
    void place(PlaceOrderRequestModel request);
}

public interface PlaceOrderOutputBoundary {
    void present(PlaceOrderResponseModel response);
    void presentFailure(PlaceOrderFailureModel failure);
}

public class PlaceOrderInteractor implements PlaceOrderInputBoundary {
    private final OrderGateway orderGateway;
    private final PaymentGateway paymentGateway;
    private final PlaceOrderOutputBoundary presenter;

    public PlaceOrderInteractor(
        OrderGateway orderGateway,
        PaymentGateway paymentGateway,
        PlaceOrderOutputBoundary presenter
    ) {
        this.orderGateway = orderGateway;
        this.paymentGateway = paymentGateway;
        this.presenter = presenter;
    }

    @Override
    public void place(PlaceOrderRequestModel request) {
        try {
            Order order = Order.create(request.customerId(), request.items());
            paymentGateway.charge(order.totalPrice(), request.paymentMethod());
            order.place();
            orderGateway.save(order);
            presenter.present(new PlaceOrderResponseModel(order.id()));
        } catch (DomainException e) {
            presenter.presentFailure(new PlaceOrderFailureModel(e.getMessage()));
        }
    }
}
```

### 6.3 Controller (input adapter)

```java
package com.example.order.adapters.controllers;

@RestController
@RequestMapping("/orders")
class OrderController {
    private final PlaceOrderInputBoundary placeOrder;
    private final PlaceOrderPresenter presenter;

    @PostMapping
    OrderViewModel create(@Valid @RequestBody PlaceOrderHttpRequest req) {
        placeOrder.place(req.toRequestModel());
        return presenter.viewModel();      // presenter 가 누적한 view model
    }
}
```

### 6.4 Presenter (output adapter)

```java
package com.example.order.adapters.presenters;

@Component
@RequestScope
class PlaceOrderPresenter implements PlaceOrderOutputBoundary {
    private OrderViewModel viewModel;

    @Override
    public void present(PlaceOrderResponseModel response) {
        this.viewModel = new OrderViewModel(
            response.orderId().value(),
            "created"
        );
    }

    @Override
    public void presentFailure(PlaceOrderFailureModel failure) {
        this.viewModel = new OrderViewModel(null, "error: " + failure.reason());
    }

    OrderViewModel viewModel() { return viewModel; }
}
```

---

## 7. Hexagonal 과의 차이

| 항목 | Hexagonal | Clean |
| --- | --- | --- |
| 명시적 계층 수 | 2-3 | 4 |
| Output Boundary | 보통 driven port 만 (반환은 정상 흐름) | Presenter 까지 명시 (단방향 흐름) |
| 폴더 구조 | domain / application / adapter | entities / usecases / adapters / frameworks |
| use case 표현 | service method | Interactor class + Input/Output Boundary interface |
| 학습 곡선 | 보통 | 큼 |

→ Clean = Hexagonal + Presenter 명시 + 4 계층 강제.

---

## 8. 비용 / 트레이드오프

| 비용 | 의미 |
| --- | --- |
| 코드량 폭증 | Use Case 당 InputBoundary / Interactor / OutputBoundary / RequestModel / ResponseModel / Presenter |
| 학습 곡선 | 신입 개발자 적응에 4-8 주 |
| 작은 변경의 PR 크기 | 매우 큼 |
| Presenter 의 stateful 처리 | request-scoped 또는 별도 인스턴스 — 복잡 |

| 이점 | 의미 |
| --- | --- |
| 매우 강한 격리 | framework migration / DB 교체 시 entity / use case 변경 0 |
| 테스트 격리 | 각 계층 단독 테스트 |
| 큰 팀 / 장기 유지보수 | 명시적 계층 / 책임 — 일관성 |
| 도메인 보존 | 다른 모든 것이 바뀌어도 entity 유지 |

---

## 9. 적용 조건

| 조건 | 적합 |
| --- | --- |
| 도메인이 매우 복잡 + 장기 유지보수 (5+ 년) | ★ 적합 |
| 대규모 팀 (10+) + 명시적 계층 필요 | ★ 적합 |
| 인프라 교체 가능성 매우 높음 | ★ 적합 |
| 단순 CRUD | ✗ over-engineering |
| MVP / startup 초기 | ✗ 비용 너무 큼 |
| 마이크로서비스 (작은 서비스) | △ Hexagonal 이 더 적합 |

→ 대부분의 백엔드는 Hexagonal 로 충분. Clean 은 매우 복잡 + 장기 유지보수 시에만.

---

## 10. 실용적 절충 — "Pragmatic Clean Architecture"

엄격한 4 동심원 + Input/Output Boundary 가 부담스러우면 다음 절충안.

| 절충 | 설명 |
| --- | --- |
| Use Case 가 직접 응답 반환 | OutputBoundary / Presenter 생략 — Hexagonal 과 비슷 |
| Entity / Use Case 동일 패키지 | 두 layer 합침 — 작은 도메인에 적합 |
| Spring annotation 도 entity 에 허용 | 학습 비용 ↓ but framework 결합 |

→ 절충하면 사실상 Hexagonal 또는 Layered. "Clean" 의 이름만 따르지 말고 가치 (격리 / dependency rule) 를 우선.

---

## 11. 검증 — ArchUnit (Java)

```java
@AnalyzeClasses(packages = "com.example.order")
class CleanArchitectureTest {

    @ArchTest
    static final ArchRule layered_dependencies =
        layeredArchitecture().consideringAllDependencies()
            .layer("Entities").definedBy("..entities..")
            .layer("UseCases").definedBy("..usecases..")
            .layer("Adapters").definedBy("..adapters..")
            .layer("Frameworks").definedBy("..frameworks..")
            .whereLayer("Entities").mayNotAccessAnyLayer()
            .whereLayer("UseCases").mayOnlyAccessLayers("Entities")
            .whereLayer("Adapters").mayOnlyAccessLayers("UseCases", "Entities")
            .whereLayer("Frameworks").mayOnlyAccessLayers("Adapters", "UseCases", "Entities");
}
```

---

## 12. 흔한 실패 모드

| 실패 | 원인 | 정정 |
| --- | --- | --- |
| Entity 가 `@Entity` / `@Column` 의존 | framework leak | 별도 ORM Entity + Mapper |
| Use Case 가 framework annotation 다수 | `@Service`, `@Transactional` 의존 | Use Case interface + 별도 구현 |
| Presenter 의 stateful 처리 어려움 | thread / request scope 혼동 | 단순화 — return 으로 충분하면 OutputBoundary 생략 |
| InputBoundary 가 1 method 만 (`void place(...)`) | over-engineering | Hexagonal 의 use case service method 가 더 적합 |
| Adapter 가 다른 Adapter 직접 import | dependency rule 위반 | Use Case / Entity 통해 |
| 모든 작은 프로젝트에 Clean 적용 | 학습 비용 / 코드량 폭주 | Layered 또는 Hexagonal 로 시작 |
| dependency rule 검증 없음 | 형식만 적용 | ArchUnit / dependency-cruiser 도입 |

---

## 13. 마이그레이션 절차 — Hexagonal → Clean

| 단계 | 작업 |
| --- | --- |
| 1 | Use Case 별 InputBoundary interface 추출 |
| 2 | Use Case 별 ResponseModel DTO 정의 |
| 3 | OutputBoundary interface 도입 (선택) |
| 4 | Presenter 분리 (선택) — 또는 Controller 가 viewModel 변환 |
| 5 | ArchUnit 으로 4 계층 의존 검증 |

→ 가치가 명확하지 않으면 Hexagonal 에서 멈춤. Clean 은 추가 가치가 비용을 정당화할 때만.

---

## 14. 관련

- [[architecture-patterns|↑ architecture-patterns]]
- [[pattern-hexagonal-architecture]]
- [[pattern-layered-architecture]]
- [[../../20-areas/computer-science/object-oriented-programming/oop-for-backend|↗ OOP for backend]]
- [[../../20-areas/computer-science/object-oriented-programming/ddd-tactical-patterns|↗ DDD tactical patterns]]
- [[../../20-areas/computer-science/object-oriented-programming/solid-principles|↗ SOLID]]
- Robert C. Martin — "Clean Architecture" (2017)
- Robert C. Martin — "The Clean Architecture" (2012, blog) — 원전
- Mark Seemann — "Code That Fits in Your Head" (2021) — 실용 가이드
- ArchUnit — https://www.archunit.org
