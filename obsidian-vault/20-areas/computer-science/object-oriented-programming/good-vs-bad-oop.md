---
title: "좋은 OOP vs 나쁜 OOP — 식별 기준 / 코드 비교"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-18T17:30:00+09:00
tags: [computer-science, oop, code-quality, comparison]
home_hub: object-oriented-programming
related:
  - "[[object-oriented-programming]]"
  - "[[concepts]]"
  - "[[solid-principles]]"
  - "[[anti-patterns]]"
  - "[[coupling-cohesion]]"
---

# 좋은 OOP vs 나쁜 OOP — 식별 기준 / 코드 비교

**[[object-oriented-programming|↑ object-oriented-programming]]**

---

## 1. 목적

본 문서는 "좋은 OOP" 와 "나쁜 OOP" 를 객관적 기준으로 식별하고 동일 도메인의 before / after 코드로 차이를 비교한다.

본 문서가 정의하는 것:
- 좋은 OOP / 나쁜 OOP 의 8 가지 식별 기준 (각각 측정 가능)
- 동일 도메인 코드의 나쁜 → 좋은 리팩토링 비교 (3 케이스)
- 식별 기준이 깨졌을 때의 결과 신호
- 좋은 OOP 평가 체크 항목

본 문서가 정의하지 않는 것:
- 원칙 정의 자체 — [[solid-principles]], [[concepts]]
- 안티패턴 카탈로그 — [[anti-patterns]]
- 결합 / 응집 종류 — [[coupling-cohesion]]

---

## 2. 평가의 전제

"좋은 OOP" 는 절대값이 아니라 다음 4 가지 trade-off 안에서의 최적이다.

| 축 | 양 끝 |
| --- | --- |
| 추상화 | 너무 적음 (모든 게 노출) ↔ 너무 많음 (인터페이스 폭발) |
| 분할 | 한 클래스에 모든 것 ↔ 클래스 1000 개 |
| 변경 가능성 | mutable 자유 ↔ 모든 게 immutable |
| 명시성 | 모든 의존 명시 ↔ 자동 wiring (magic) |

본 문서의 모든 평가는 "현재 도메인의 변경 빈도 / 팀 규모 / 테스트 가능성" 의 맥락 안에서 적용된다.

---

## 3. 좋은 OOP 의 8 가지 식별 기준

| # | 기준 | 측정 |
| --- | --- | --- |
| 1 | **명확한 책임** | 한 클래스의 method 들이 같은 데이터 / 같은 변경 actor 를 다룬다 (cohesion) |
| 2 | **낮은 결합** | 한 클래스 변경이 다른 클래스의 수정 / 컴파일을 유발하지 않는다 (coupling) |
| 3 | **선언적 행위 (tell, don't ask)** | 클라이언트가 객체에게 일을 시킨다. 상태를 묻고 외부에서 결정하지 않는다 |
| 4 | **invariant 강제** | 객체가 invalid state 로 진입할 수 없도록 method 가 검증한다 |
| 5 | **대체 가능성 (LSP)** | 같은 인터페이스의 다른 구현으로 교체 시 호출자 변경 0 |
| 6 | **변경의 지역성** | 요구 변경 1 건이 수정하는 클래스 수 ≤ 2-3 개 |
| 7 | **테스트 용이성** | unit test 가 외부 의존 없이 객체 단위로 작성 가능 |
| 8 | **명시적 의존** | 객체가 필요로 하는 것을 constructor / parameter 로 받는다 (전역 / static 의존 X) |

→ 8 항목이 모두 동시에 충족될 수는 없다. 도메인 / 단계에 따라 우선순위가 다르다.

---

## 4. 나쁜 OOP 의 식별 신호

위 8 기준의 위반은 다음 신호로 관찰된다.

| 신호 | 위반 기준 |
| --- | --- |
| 클래스 이름이 Manager / Helper / Util / Processor | (1) 책임 모호 |
| getter / setter 만 있고 행위 method 없음 | (3) tell, don't ask 위반 (anemic) |
| 1 요구 변경이 5+ 파일 PR | (6) 변경 cascade — coupling 높음 |
| 모든 method 가 static | OOP 가 아님 (절차적) |
| 외부 객체의 상태를 직접 변경 (`order.getItems().add(...)`) | (3) Law of Demeter 위반 |
| Mock 작성에 의존 객체 10+ 개 | (2)(8) 결합 높음 + 의존 폭주 |
| `instanceof` 분기로 동작 분기 | (5) polymorphism 무력화 |
| 같은 데이터의 invariant 검증이 여러 곳에 반복 | (4) encapsulation 위반 |
| `new ConcreteClass()` 가 비즈니스 로직 안에 | (8) 의존 inline 생성 |
| 전역 singleton / static state 접근 | (8) hidden dependency |

---

## 5. 비교 케이스 1 — 주문 처리

### 5.1 나쁜 OOP

```java
class Order {
    public List<Item> items;
    public String status;
    public long totalPrice;
    public Customer customer;
    // 모든 field public
}

class OrderManager {
    public void processOrder(Order o) {
        // o 의 상태를 외부에서 검증
        if (o.status == null || o.status.equals("CANCELLED")) {
            throw new RuntimeException("invalid");
        }

        // 가격 계산을 외부에서
        long total = 0;
        for (Item it : o.items) {
            total += it.price * it.quantity;
        }
        o.totalPrice = total;     // 직접 mutate

        // 할인 적용을 외부에서
        if (o.customer.tier.equals("VIP")) {
            o.totalPrice = (long)(o.totalPrice * 0.9);
        }

        // 상태 전이를 외부에서
        o.status = "PROCESSED";

        // DB / Email 도 같은 메서드
        new MySqlOrderRepository().save(o);
        new SmtpEmailSender().send(o.customer.email, "...");
    }
}
```

| 위반 항목 |
| --- |
| (1) OrderManager 가 검증 / 가격 / 할인 / 상태 전이 / 영속 / 알림 모두 책임 |
| (3) Order 가 anemic — OrderManager 가 Order 의 상태를 묻고 외부에서 변경 |
| (4) status / totalPrice 가 외부에서 임의 변경 가능 |
| (8) MySqlOrderRepository / SmtpEmailSender 를 inline `new` |

### 5.2 좋은 OOP

```java
// Order — 자기 invariant 와 행위를 보유
class Order {
    private final OrderId id;
    private final Customer customer;
    private final List<Item> items;
    private OrderStatus status;
    private Money totalPrice;

    Order(OrderId id, Customer customer, List<Item> items) {
        if (items.isEmpty()) throw new IllegalArgumentException("empty items");
        this.id = id;
        this.customer = customer;
        this.items = List.copyOf(items);
        this.status = OrderStatus.CREATED;
        this.totalPrice = calculateTotal();
    }

    private Money calculateTotal() {
        Money sum = items.stream()
            .map(Item::subtotal)
            .reduce(Money.ZERO, Money::plus);
        return customer.applyDiscount(sum);    // 할인은 Customer 가 안다
    }

    void process() {
        if (status != OrderStatus.CREATED) {
            throw new IllegalStateException("cannot process " + status);
        }
        status = OrderStatus.PROCESSED;
    }

    OrderId id()          { return id; }
    Money   totalPrice()  { return totalPrice; }
    OrderStatus status()  { return status; }
}

// Customer — 자기의 할인 정책을 안다
class Customer {
    private final CustomerId id;
    private final String email;
    private final Tier tier;

    Money applyDiscount(Money base) {
        return tier == Tier.VIP ? base.multiply(0.9) : base;
    }

    String email() { return email; }
}

// Use case — 협력 조율만
class PlaceOrder {
    private final OrderRepository repo;
    private final EmailSender sender;

    PlaceOrder(OrderRepository repo, EmailSender sender) {
        this.repo = repo;
        this.sender = sender;
    }

    void place(Order order) {
        order.process();
        repo.save(order);
        sender.send(order.customer().email(), "주문 완료");
    }
}
```

| 충족 항목 |
| --- |
| (1) Order = 자기 상태 / 행위 / invariant. PlaceOrder = 협력 조율. |
| (3) `order.process()` — tell. 외부가 status 를 set 하지 않음. |
| (4) constructor + process() 안에서 invariant 강제. |
| (5) OrderRepository / EmailSender 인터페이스 — Mysql / Stub 교체 자유. |
| (8) constructor 주입. |

---

## 6. 비교 케이스 2 — 결제 분기

### 6.1 나쁜 OOP

```java
class PaymentHandler {
    public void pay(String method, long amount, String accountInfo) {
        if (method.equals("CARD")) {
            // 카드 처리 100 줄
        } else if (method.equals("BANK")) {
            // 계좌이체 처리 80 줄
        } else if (method.equals("PAYPAL")) {
            // PayPal 처리 60 줄
        } else if (method.equals("KAKAO")) {
            // 카카오페이 처리 70 줄
        }
        throw new RuntimeException("unknown method");
    }
}
```

| 위반 항목 |
| --- |
| (1) 4 결제 수단의 책임이 한 클래스 |
| (5) polymorphism 무력화 — 새 수단마다 if 추가 (OCP 위반) |
| (6) 수단 추가 → PaymentHandler 1 파일 + 호출자 4 곳 수정 |

### 6.2 좋은 OOP

```java
sealed interface PaymentMethod
    permits CardPayment, BankPayment, PaypalPayment, KakaoPayment {

    PaymentResult charge(Money amount);
}

record CardPayment(CardNumber number, Cvc cvc) implements PaymentMethod {
    public PaymentResult charge(Money amount) { /* 카드 처리 */ }
}
record BankPayment(BankAccount account) implements PaymentMethod {
    public PaymentResult charge(Money amount) { /* 계좌이체 */ }
}
record PaypalPayment(String paypalEmail) implements PaymentMethod {
    public PaymentResult charge(Money amount) { /* */ }
}
record KakaoPayment(String kakaoUserId) implements PaymentMethod {
    public PaymentResult charge(Money amount) { /* */ }
}

class Checkout {
    PaymentResult checkout(Order o, PaymentMethod method) {
        return method.charge(o.totalPrice());
    }
}
```

| 충족 항목 |
| --- |
| (1) 각 결제 수단이 자기 책임 |
| (5) Checkout 은 PaymentMethod 만 안다. 새 수단 추가 시 Checkout 변경 0 (OCP). |
| (6) 수단 추가 = 새 record 1 개 + (필요 시) `sealed` 의 `permits` 1 행 추가. |
| Java 17+ `sealed` 가 exhaustive switch 보장. |

---

## 7. 비교 케이스 3 — 알림 발송

### 7.1 나쁜 OOP

```java
class NotificationService {
    public void send(User user, String message) {
        // user.preferences 의 값을 읽어 외부에서 분기
        if (user.getPreferences().getNotificationType().equals("EMAIL")) {
            new SmtpClient().send(user.getEmail(), message);
        } else if (user.getPreferences().getNotificationType().equals("SMS")) {
            new TwilioClient().send(user.getPhoneNumber(), message);
        } else if (user.getPreferences().getNotificationType().equals("PUSH")) {
            new FcmClient().send(user.getDeviceToken(), message);
        }
    }
}
```

| 위반 항목 |
| --- |
| (3) `user.getPreferences().getNotificationType()` — Law of Demeter 위반 (chain 3 단계) |
| (5) polymorphism 무력화 — instanceof 형태의 string 분기 |
| (8) SmtpClient / TwilioClient / FcmClient inline `new` |

### 7.2 좋은 OOP

```java
interface NotificationChannel {
    void send(User user, String message);
}

class EmailChannel implements NotificationChannel {
    private final EmailSender sender;
    public EmailChannel(EmailSender s) { this.sender = s; }
    public void send(User u, String m) { sender.send(u.email(), m); }
}
class SmsChannel implements NotificationChannel { /* */ }
class PushChannel implements NotificationChannel { /* */ }

// User 가 자기 채널을 선택
class User {
    private final NotificationChannelType preferredChannel;
    NotificationChannelType preferredChannel() { return preferredChannel; }
    // email / phone / deviceToken 는 외부에 노출하지 않고 channel 이 알게 함
}

class NotificationService {
    private final Map<NotificationChannelType, NotificationChannel> channels;
    NotificationService(Map<NotificationChannelType, NotificationChannel> channels) {
        this.channels = Map.copyOf(channels);
    }
    void send(User u, String m) {
        channels.get(u.preferredChannel()).send(u, m);
    }
}
```

| 충족 항목 |
| --- |
| (3) chain 제거. User 가 preferredChannel 만 노출. |
| (5) NotificationChannel interface 로 polymorphism. |
| (8) constructor 주입. test 에서 InMemoryChannel 로 교체 가능. |

---

## 8. 좋은 OOP 평가 체크 항목

다음 항목이 새 코드 / PR 리뷰 시 1 차 점검 대상.

| 항목 | 점검 |
| --- | --- |
| 클래스 이름이 도메인 용어인가 (Manager / Helper / Util 금지) |  |
| 클래스의 method 들이 같은 actor 의 같은 데이터를 다루는가 |  |
| public field 또는 getter/setter 만 있는 anemic 객체가 없는가 |  |
| invariant 가 constructor + method 안에서만 강제되는가 |  |
| 외부 객체의 chain (`a.b().c().d()`) 가 없는가 |  |
| polymorphism 으로 표현 가능한 분기에 instanceof / string switch 가 없는가 |  |
| 객체가 의존하는 것이 모두 constructor / parameter 로 들어오는가 |  |
| 인터페이스를 통해 의존하는가 (구체 class 직접 의존 X — 단, 두 번째 구현 가능성 있는 경우) |  |
| unit test 가 외부 의존 없이 작성 가능한가 |  |
| 같은 도메인 변경 1 건이 수정하는 클래스 수가 2-3 개 이하인가 |  |

---

## 9. 결과 신호 — 나쁜 OOP 가 production 에 도달했을 때

| 신호 | 원인 |
| --- | --- |
| 새 기능 추가 시 PR 의 파일 수가 매번 증가 | coupling 누적 |
| 같은 버그 패턴 (`null` / `negative balance`) 이 반복 등장 | invariant 가 한 곳에 모이지 않음 |
| 테스트 실행 시 외부 DB / 외부 API 필수 | DIP 위반 — 인프라가 비즈니스 로직에 내장 |
| 한 사람만 특정 모듈을 수정 가능 (bus factor 1) | class 가 너무 크고 책임이 혼재 |
| 리팩토링 PR 이 항상 reject (위험) | 결합이 너무 높아 변경 위험 큼 |
| 신입 onboarding 시 "여기는 함부로 건들지 마라" 영역 다수 | 변경 위험의 시스템화 |

→ 이 신호가 누적되면 [[anti-patterns]] 의 카탈로그를 참고해 리팩토링 우선순위 정하기.

---

## 10. 참고

- [[object-oriented-programming|↑ OOP hub]]
- [[concepts]]
- [[solid-principles]]
- [[anti-patterns]]
- [[coupling-cohesion]]
- [[composition-over-inheritance]]
- [[oop-for-backend]]
- Martin Fowler — "Refactoring" 2판 (2018) — code smell + 리팩토링 카탈로그
- Eric Evans — "Domain-Driven Design" (2003) — anemic vs rich domain model
- Sandi Metz — "Practical Object-Oriented Design in Ruby" (POODR)
- John Ousterhout — "A Philosophy of Software Design" (2018) — depth of module
