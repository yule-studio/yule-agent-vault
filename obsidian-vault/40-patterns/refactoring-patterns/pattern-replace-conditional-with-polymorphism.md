---
title: "pattern-replace-conditional-with-polymorphism — switch / instanceof → polymorphism"
kind: knowledge
project: patterns
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-19T00:00:00+09:00
tags: [patterns, refactoring, polymorphism, oop, ocp]
home_hub: refactoring-patterns
related:
  - "[[refactoring-patterns]]"
  - "[[../../20-areas/computer-science/object-oriented-programming/solid-principles]]"
  - "[[../../20-areas/computer-science/object-oriented-programming/concepts]]"
  - "[[../../20-areas/computer-science/object-oriented-programming/anti-patterns]]"
  - "[[../../20-areas/computer-science/design-patterns/strategy/strategy]]"
---

# pattern-replace-conditional-with-polymorphism — switch / instanceof → polymorphism

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-19 | engineering-agent/tech-lead | 최초 정리 |

**[[refactoring-patterns|↑ refactoring-patterns]]**

---

## 1. 의도

type / status / 모드별로 다른 동작을 표현하는 `switch` / `if-else` / `instanceof` 분기를 **polymorphism (subtype 또는 strategy)** 으로 교체한다. 새 case 추가 시 기존 분기 코드를 수정하지 않게 만든다 (OCP 충족).

원전: Martin Fowler — "Refactoring" 2판 (2018) "Replace Conditional with Polymorphism".

---

## 2. 적용 신호

다음 신호 중 1 개 이상이면 후보.

| 신호 | 의미 |
| --- | --- |
| 같은 형태의 `switch (type)` 가 여러 곳 반복 | 새 case 추가 시 shotgun surgery |
| `if (x instanceof A) ... else if (x instanceof B) ...` | polymorphism 무력화 |
| enum 의 모든 값에 대한 case 가 여러 method 에서 반복 | type 분기 확산 |
| string 비교 (`type.equals("CARD")`) 로 분기 | type-safety 0 + magic string |
| 새 type 추가 시 5+ 파일 PR | OCP 위반 |

---

## 3. 적용 회피 시점

다음 경우는 polymorphism 변환이 오히려 비효율.

| 회피 조건 | 이유 |
| --- | --- |
| case 가 2 개 + 변경 가능성 낮음 | 단순 if-else 가 더 읽기 쉬움 |
| case 별 동작이 거의 같음 + 차이 작음 | parameter 차이만으로 충분 |
| performance critical path 의 미세한 dispatch overhead | (드문 경우) — 실측 후 결정 |
| sealed + exhaustive pattern matching 충분 | Java 21+ / Kotlin 의 `sealed when` 으로 컴파일 안전성 + 새 type 추가 시 컴파일 에러로 가이드 |

→ Java 21+ 의 `sealed + pattern matching` 은 polymorphism 의 대안. 두 접근 모두 OCP / type-safety 충족.

---

## 4. 절차 — 절차형 switch → polymorphism

### 4.1 before

```java
class Payment {
    String type;
    double amount;
    String cardNumber;
    String bankAccount;
    String paypalEmail;
}

class PaymentProcessor {
    void process(Payment p) {
        if (p.type.equals("CARD")) {
            // 카드 처리 50 줄
            chargeCard(p.cardNumber, p.amount);
        } else if (p.type.equals("BANK")) {
            // 계좌 처리 40 줄
            transferFromBank(p.bankAccount, p.amount);
        } else if (p.type.equals("PAYPAL")) {
            // PayPal 처리 30 줄
            chargePaypal(p.paypalEmail, p.amount);
        } else {
            throw new IllegalArgumentException("unknown type: " + p.type);
        }
    }

    double calculateFee(Payment p) {
        if (p.type.equals("CARD")) return p.amount * 0.03;
        else if (p.type.equals("BANK")) return 500;
        else if (p.type.equals("PAYPAL")) return p.amount * 0.05;
        throw new IllegalArgumentException();
    }
}
```

→ 동일한 type 분기가 2 method (process, calculateFee) 에 반복. 새 결제 수단 추가 시 두 method + Payment 모두 수정.

### 4.2 step 1 — Extract Class 로 subtype 후보 분리

```java
sealed interface Payment permits CardPayment, BankPayment, PaypalPayment {
    double amount();
}

record CardPayment(double amount, String cardNumber) implements Payment { }
record BankPayment(double amount, String bankAccount) implements Payment { }
record PaypalPayment(double amount, String paypalEmail) implements Payment { }
```

### 4.3 step 2 — switch 의 분기 동작을 subtype 의 method 로 이동 (Move Method)

```java
sealed interface Payment permits CardPayment, BankPayment, PaypalPayment {
    double amount();
    void process();
    double calculateFee();
}

record CardPayment(double amount, String cardNumber) implements Payment {
    public void process() { /* 카드 처리 50 줄 */ }
    public double calculateFee() { return amount * 0.03; }
}

record BankPayment(double amount, String bankAccount) implements Payment {
    public void process() { /* 계좌 처리 40 줄 */ }
    public double calculateFee() { return 500; }
}

record PaypalPayment(double amount, String paypalEmail) implements Payment {
    public void process() { /* PayPal 처리 30 줄 */ }
    public double calculateFee() { return amount * 0.05; }
}
```

### 4.4 step 3 — 호출자 (PaymentProcessor) 단순화

```java
class PaymentProcessor {
    void process(Payment p) { p.process(); }
    double calculateFee(Payment p) { return p.calculateFee(); }
}
```

→ 새 결제 수단 추가 = Payment 의 새 record 1 개 + `sealed` 의 `permits` 1 행 추가. `PaymentProcessor` 변경 0.

---

## 5. 절차 — 절차형 enum switch → polymorphism

### 5.1 before

```java
enum OrderStatus { CREATED, PLACED, SHIPPED, DELIVERED, CANCELLED }

class OrderService {
    boolean canBeCancelled(Order order) {
        return switch (order.getStatus()) {
            case CREATED, PLACED -> true;
            case SHIPPED -> false;
            case DELIVERED, CANCELLED -> false;
        };
    }

    String userFacingLabel(Order order) {
        return switch (order.getStatus()) {
            case CREATED -> "주문 작성 중";
            case PLACED -> "주문 완료";
            case SHIPPED -> "배송 중";
            case DELIVERED -> "배송 완료";
            case CANCELLED -> "취소됨";
        };
    }
}
```

→ enum value 의 동작이 외부 (OrderService) 에 흩어짐. 새 status 추가 시 모든 switch 갱신.

### 5.2 after — enum + abstract method (Java 의 enum 의 강력한 기능)

```java
enum OrderStatus {
    CREATED("주문 작성 중") {
        public boolean canBeCancelled() { return true; }
    },
    PLACED("주문 완료") {
        public boolean canBeCancelled() { return true; }
    },
    SHIPPED("배송 중") {
        public boolean canBeCancelled() { return false; }
    },
    DELIVERED("배송 완료") {
        public boolean canBeCancelled() { return false; }
    },
    CANCELLED("취소됨") {
        public boolean canBeCancelled() { return false; }
    };

    private final String label;
    OrderStatus(String label) { this.label = label; }

    public String label() { return label; }
    public abstract boolean canBeCancelled();
}

class OrderService {
    boolean canBeCancelled(Order order) { return order.getStatus().canBeCancelled(); }
    String userFacingLabel(Order order) { return order.getStatus().label(); }
}
```

→ enum value 자체에 동작 결합. 새 status 추가 시 enum 만 수정. `abstract method` 가 누락 시 컴파일 에러로 가이드.

---

## 6. 절차 — instanceof chain → visitor pattern (대규모)

### 6.1 before

```java
abstract class Shape { }
class Circle extends Shape { double radius; }
class Square extends Shape { double side; }
class Rectangle extends Shape { double w, h; }

class AreaCalculator {
    double area(Shape s) {
        if (s instanceof Circle c) return Math.PI * c.radius * c.radius;
        if (s instanceof Square sq) return sq.side * sq.side;
        if (s instanceof Rectangle r) return r.w * r.h;
        throw new IllegalArgumentException();
    }
}
```

### 6.2 after — Java 21 sealed + pattern matching

```java
sealed interface Shape permits Circle, Square, Rectangle { }
record Circle(double radius) implements Shape { }
record Square(double side) implements Shape { }
record Rectangle(double w, double h) implements Shape { }

class AreaCalculator {
    double area(Shape s) {
        return switch (s) {                            // exhaustive — 컴파일러가 모든 case 검증
            case Circle c -> Math.PI * c.radius() * c.radius();
            case Square sq -> sq.side() * sq.side();
            case Rectangle r -> r.w() * r.h();
        };
    }
}
```

| 차이 | 효과 |
| --- | --- |
| `sealed` | 모든 subtype 이 컴파일러에 알려짐 |
| pattern matching switch | 새 subtype 추가 시 모든 switch 가 컴파일 에러 |
| record | boilerplate 0 |

→ Java 21+ / Scala 의 sealed pattern matching 은 polymorphism 의 강력한 대안. 외부 인터페이스에 method 추가 없이 새 동작 (visitor) 추가 가능.

---

## 7. 변환 단계별 안전 절차

| 단계 | 작업 | 안전 검증 |
| --- | --- | --- |
| 1 | 모든 switch / instanceof 분기를 식별 | grep / IDE 검색 |
| 2 | 외부 동작 검증 테스트 작성 (없으면 characterization test) | 통과 확인 |
| 3 | type 별 subtype class / record 정의 | 컴파일 확인 |
| 4 | switch 의 동작 1 개를 subtype method 로 이동 (Move Method) | 테스트 통과 |
| 5 | 호출자 switch 를 polymorphism 호출로 교체 | 테스트 통과 |
| 6 | 4-5 를 모든 분기에 반복 | 테스트 통과 |
| 7 | 사용 안 하는 type 필드 / 분기 제거 | 컴파일 + 테스트 |
| 8 | sealed (Java 17+) / final (안전 보장) | 새 case 추가 시 컴파일 에러로 가이드 |

→ 한 PR 에 1-2 분기 만 — 작은 step 으로 review 가능.

---

## 8. 효과 측정

| 지표 | 측정 |
| --- | --- |
| 같은 type 분기의 반복 횟수 | 감소 (0 목표) |
| 새 type 추가 PR 의 수정 파일 수 | 감소 (1-2 개) |
| cyclomatic complexity | 감소 |
| 새 case 누락 버그 (default 분기 도달) | 감소 (sealed 면 0) |
| 클래스 LCOM | 감소 |

---

## 9. 흔한 실패 모드

| 실패 | 원인 | 정정 |
| --- | --- | --- |
| polymorphism 변환 후 subtype 이 비대 (1000 줄) | switch case 가 비대했었던 것을 그대로 이동 | 각 subtype 안에서 추가 Extract Method |
| 모든 분기를 polymorphism 으로 변환 | 단순 if-else 까지 강제 | case 2 + 변경 가능성 낮음은 회피 |
| subtype 간 공통 코드가 중복 | template method / composition 부재 | Pull Up Method / 컴포지션 도입 |
| sealed 의 case 가 매우 많음 (20+) | 잘못된 도메인 모델 | 도메인 재분류 — Aggregate 분리 |
| Visitor 적용으로 boilerplate 폭주 | visitor 가 필요 없는 경우 | sealed + pattern matching |
| Java 8 / Python 3 에서 sealed 부재 | 언어 한정 | abstract method / interface + 런타임 검증 |

---

## 10. 다른 패턴과의 관계

| 패턴 | 관계 |
| --- | --- |
| **Strategy** (GoF) | 본 패턴이 적용된 결과의 한 형태 (행위의 컴포지션) |
| **State** (GoF) | 상태별 행위 polymorphism — 본 패턴의 특수 사례 |
| **Visitor** (GoF) | sealed 가 없을 때 같은 효과 |
| **Replace Type Code with Subclasses** (Fowler) | type 필드를 subclass 로 — 본 패턴의 기반 |
| **Move Method** (Fowler) | 본 패턴의 step 4 |
| **Pull Up Method** (Fowler) | 공통 코드를 부모로 (본 패턴 후) |

상세: [[../../20-areas/computer-science/design-patterns/strategy/strategy|↗ Strategy]], [[../../20-areas/computer-science/design-patterns/state/state|↗ State]] (존재 시), [[../../20-areas/computer-science/design-patterns/visitor/visitor|↗ Visitor]].

---

## 11. 적용 검증 — 코드 리뷰 체크

PR 리뷰 시 다음 확인.

| 항목 | 점검 |
| --- | --- |
| switch / instanceof chain 이 1 곳에만 (factory) 인가 |  |
| 같은 type 의 동작 추가 시 1 파일 (subtype) 만 수정인가 |  |
| sealed / final 로 새 subtype 등록 통제되는가 |  |
| 모든 case 에 대한 컴파일 안전성 (exhaustive switch) 보장되는가 |  |
| subtype 의 method 가 작은가 (한 책임) |  |
| common 코드가 부모 / interface default 로 추출되었는가 |  |

---

## 12. 관련

- [[refactoring-patterns|↑ refactoring-patterns]]
- [[../../20-areas/computer-science/object-oriented-programming/solid-principles|↗ OCP]]
- [[../../20-areas/computer-science/object-oriented-programming/concepts|↗ polymorphism]] §9
- [[../../20-areas/computer-science/object-oriented-programming/anti-patterns|↗ instanceof 분기 안티패턴]]
- [[../../20-areas/computer-science/object-oriented-programming/good-vs-bad-oop|↗ 결제 분기 케이스]]
- [[../../20-areas/computer-science/design-patterns/strategy/strategy|↗ Strategy 패턴]]
- Martin Fowler — "Refactoring" 2판 (2018) §10 — Replace Conditional with Polymorphism
- Joshua Bloch — "Effective Java" Item 34 — enum 의 method 활용
- JEP 409 — Sealed Classes (Java 17)
- JEP 441 — Pattern Matching for switch (Java 21)
