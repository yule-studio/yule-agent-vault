---
title: "tell, don't ask + Law of Demeter — OOP 통신 원칙"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-18T20:00:00+09:00
tags: [computer-science, oop, principles, communication, encapsulation]
home_hub: object-oriented-programming
related:
  - "[[object-oriented-programming]]"
  - "[[concepts]]"
  - "[[solid-principles]]"
  - "[[good-vs-bad-oop]]"
  - "[[anti-patterns]]"
  - "[[value-objects]]"
---

# tell, don't ask + Law of Demeter — OOP 통신 원칙

**[[object-oriented-programming|↑ object-oriented-programming]]**

---

## 1. 목적

본 문서는 OOP 객체 간 통신의 두 가지 핵심 원칙 — "Tell, Don't Ask" 와 "Law of Demeter (LoD)" — 를 정의하고, 두 원칙이 캡슐화 / 결합도와 어떻게 연결되는지 정리한다.

본 문서가 정의하는 것:
- Tell, Don't Ask 의 정의 / 위반 신호 / 적용
- Law of Demeter 의 정의 / 5 규칙 / 위반 신호 / 적용
- 두 원칙의 관계와 차이
- 위반의 결과 (결합도 증가, encapsulation 깨짐, 변경 cascade)
- 적용의 한계 (조회 / 보고서 / DTO 경계)

본 문서가 정의하지 않는 것:
- 4 pillar — [[concepts]]
- SOLID — [[solid-principles]]
- 결합 / 응집 분류 — [[coupling-cohesion]]

---

## 2. 범위

| 구분 | 포함 |
| --- | --- |
| 대상 | 객체 간 메시지 호출 코드 |
| 적용 | 도메인 모델 / Service / Use Case |
| 제외 | 단순 DTO / 조회 응답 (read model) — 의도된 예외 |
| 제외 | 함수형 코드의 변환 chain (`stream().map().filter().collect()`) — 다른 의미 |

---

## 3. 용어

| 용어 | 정의 |
| --- | --- |
| **Tell, Don't Ask** | 객체에게 일을 시키지, 상태를 묻고 외부에서 결정하지 말라는 원칙. |
| **Law of Demeter (LoD)** | "최소 지식의 원칙". 한 method 가 다른 객체의 직접 친구의 친구의 친구 ... 와 대화하지 말라는 원칙. |
| **train wreck** | LoD 위반의 표면 신호. `a.getB().getC().getD().doSomething()` 같은 chain. |
| **demeter friend** | 한 method 가 직접 대화해도 되는 객체 (자기 자신, 인자, 자기가 만든 객체, 자기 field). |
| **getter chain** | getter 의 연속 호출. encapsulation 위반의 직접 증거. |

---

## 4. Tell, Don't Ask

### 4.1 정의

> "Procedural code gets information then makes decisions. Object-oriented code tells objects to do things."
> — Alec Sharp / Andy Hunt

객체는 자신의 데이터를 외부에 노출하지 않고, 클라이언트는 객체의 데이터를 묻지 않고 의도를 전달한다.

### 4.2 위반 신호

| 신호 | 의미 |
| --- | --- |
| getter 호출 후 외부에서 조건 분기 | 객체의 결정을 외부가 함 |
| 같은 검증 로직이 객체 외부 여러 곳에 반복 | 결정 책임이 분산 |
| 외부가 객체의 setter 로 직접 state 변경 | 객체의 invariant 가 외부 책임 |
| `if (account.getStatus() == ACTIVE) account.setStatus(SUSPENDED)` 같은 코드 | ask → decide → tell 의 3 단계 |
| Service 가 객체의 getter 만으로 비즈니스 로직 수행 | anemic 객체 + procedural service |

### 4.3 위반 — before

```java
class CartService {
    void checkout(Cart cart, Coupon coupon) {
        // ask 1
        if (cart.getItems().isEmpty()) {
            throw new EmptyCartException();
        }
        // ask 2
        if (cart.getTotalPrice() < coupon.getMinAmount()) {
            throw new CouponNotApplicableException();
        }
        // decide
        long discounted = cart.getTotalPrice() - coupon.getDiscountAmount();
        // tell (마지막에 한 줄)
        cart.setFinalPrice(discounted);
        cart.setStatus("CHECKED_OUT");
    }
}
```

→ CartService 가 Cart 의 모든 결정을 함. Cart 는 데이터 백 (DTO).

### 4.4 적용 — after

```java
class Cart {
    private final List<CartItem> items;
    private Money totalPrice;
    private Money finalPrice;
    private CartStatus status;

    void checkout(Coupon coupon) {
        if (items.isEmpty()) throw new EmptyCartException();
        if (!coupon.isApplicableTo(totalPrice)) throw new CouponNotApplicableException();
        finalPrice = coupon.applyTo(totalPrice);
        status = CartStatus.CHECKED_OUT;
    }
}

class Coupon {
    private final Money minAmount;
    private final Money discountAmount;

    boolean isApplicableTo(Money total) { return total.isGreaterThanOrEqual(minAmount); }
    Money applyTo(Money total)         { return total.minus(discountAmount); }
}

class CartService {
    void checkout(Cart cart, Coupon coupon) {
        cart.checkout(coupon);
    }
}
```

| 결과 | 효과 |
| --- | --- |
| Cart 의 invariant 가 Cart 안 | 외부가 invalid state 만들 수 없음 |
| Coupon 의 적용 규칙이 Coupon 안 | 쿠폰 정책 변경 시 1 곳만 |
| CartService 가 협력 조율만 | 새 결제 단계 추가 시 영향 최소 |

### 4.5 적용 절차

1. getter 후 외부 분기 식별
2. 그 분기 로직을 객체 안의 새 method 로 이동 (Move Method)
3. getter 호출 제거 — 객체에게 직접 명령
4. 외부 코드가 객체의 의도된 method 만 호출하는지 확인

---

## 5. Law of Demeter (LoD)

### 5.1 정의 — "한 객체는 자기 친구하고만 대화한다"

한 method `m` 안에서 다음 5 종류의 객체에게만 메시지를 보낼 수 있다.

| 종류 | 예 |
| --- | --- |
| 1 | 자기 자신 (this) | `this.helper()` |
| 2 | method 의 인자 | `arg.compute()` |
| 3 | 자기가 생성한 객체 | `new Foo().bar()` |
| 4 | 자기 field | `this.field.do()` |
| 5 | 정적 method 가 반환한 객체 (특정 해석에서 포함) | (선택적) |

→ "method 가 반환한 객체의 method 호출" 은 LoD 위반 (단, 같은 type 의 fluent API 는 예외).

### 5.2 train wreck — 위반 신호

```java
boolean canPay = order.getCustomer().getWallet().getBalance().isGreaterThan(price);
// chain 4 단계 — order 가 Customer / Wallet / Money 모두 노출
```

체인이 길수록 다음 위험.

| 위험 | 결과 |
| --- | --- |
| Customer 의 wallet 변경 (null 가능) | 호출자 NPE |
| Customer 의 내부 구조 변경 (`wallet` 제거) | 호출자 컴파일 에러 |
| 검증 로직 누락 | 호출자가 모를 수도 |
| 같은 chain 이 여러 곳 반복 | 캡슐화 0 |

### 5.3 위반 — before

```java
class CheckoutService {
    boolean canPlace(Order order, Money price) {
        return order.getCustomer().getWallet().getBalance().isGreaterThan(price);
    }
}
```

### 5.4 적용 — after

```java
class Order {
    boolean canAfford(Money price) {
        return customer.canAfford(price);   // delegate
    }
}

class Customer {
    boolean canAfford(Money price) {
        return wallet.canAfford(price);     // delegate
    }
}

class Wallet {
    boolean canAfford(Money price) {
        return balance.isGreaterThanOrEqual(price);
    }
}

class CheckoutService {
    boolean canPlace(Order order, Money price) {
        return order.canAfford(price);
    }
}
```

| 결과 | 효과 |
| --- | --- |
| 각 객체가 자기 책임을 노출 | encapsulation |
| Customer 가 wallet 을 다른 구조로 바꿔도 영향 0 | Customer.canAfford 만 수정 |
| chain 0 단계 | LoD 충족 |

### 5.5 LoD 와 fluent API 의 차이

```java
// LoD 위반?
String result = stringBuilder.append("a").append("b").append("c").toString();
```

→ fluent API 는 같은 type 의 self 를 반환 — `append` 가 `this` 반환 — LoD 위반 아님. 같은 객체에 연속 메시지.

→ 위반은 **서로 다른 객체의 chain**.

---

## 6. 두 원칙의 관계

| 항목 | Tell, Don't Ask | Law of Demeter |
| --- | --- | --- |
| 초점 | "묻지 말고 시켜라" — 의도 전달 | "친구하고만 대화" — chain 금지 |
| 위반 신호 | getter + 외부 분기 | a.b().c().d() chain |
| 결합도 영향 | 외부 결정 로직 → 객체 결정 | chain 단계마다 결합 ↑ |
| 캡슐화 영향 | 객체가 자기 state 보호 | 객체가 자기 친구의 친구 노출 안 함 |
| 적용 | Move Method, Hide Delegate | Hide Delegate |

→ 두 원칙은 **encapsulation 의 두 측면**. Tell, Don't Ask 가 깨지면 LoD 도 흔히 깨지며 그 반대도 같다.

---

## 7. Hide Delegate — 두 원칙의 공통 리팩토링

### 7.1 정의

객체가 내부의 다른 객체를 외부에 노출하지 않고, 필요한 method 를 자기 자신의 method 로 wrap 한다.

### 7.2 절차

| 단계 | 작업 |
| --- | --- |
| 1 | chain 또는 ask-decide-tell 코드 식별 |
| 2 | chain 의 의도를 한 단어로 명명 (`canAfford`, `applyDiscount`) |
| 3 | 첫 객체의 새 method 로 추출 |
| 4 | 두 번째 객체에도 같은 method 추가 (delegate) |
| 5 | chain / ask-decide-tell 코드 → 새 method 호출로 교체 |
| 6 | 더 이상 사용 안 되는 getter 제거 (선택)

---

## 8. 적용의 한계

두 원칙을 절대 규칙으로 적용하면 다음 영역에서 비효율.

| 영역 | 이유 |
| --- | --- |
| **DTO / Response model** | API 응답은 모든 field 가 노출되어야 함 — getter 정상 |
| **read model / 보고서** | 조회 시 객체 상태를 외부가 봐야 — getter 정상 |
| **테스트 assertion** | `assertThat(account.balance()).isEqualTo(100)` — getter 필요 |
| **로깅 / 직렬화** | 객체 외부에서 state 읽기 — getter / serializer |
| **단순 값 객체 (Money / Date)** | value 자체가 외부 사용 의도 — getter 정상 |

→ 두 원칙은 **도메인 객체의 비즈니스 결정** 에 적용. 데이터 transfer 객체에는 적용하지 않는다.

---

## 9. 두 원칙의 측정

| 측정 | 의미 |
| --- | --- |
| 한 method 안 `.` chain ≥ 3 단계 수 | LoD 위반 후보 |
| 한 class 의 public getter 수 | 정보 노출 정도 (낮을수록 encapsulation ↑) |
| 한 PR 에서 객체의 setter / getter 호출 변경 빈도 | tell-don't-ask 충족 정도 |
| Service 클래스의 평균 method 길이 | anemic 객체 신호 — 길면 객체에 행위 이동 후보 |

---

## 10. 흔한 실패 모드

| 실패 | 원인 | 정정 |
| --- | --- | --- |
| getter 만 사용 후 외부 분기 누적 | tell-don't-ask 위반 | Move Method — 결정을 객체 안으로 |
| chain 3-4 단계 흔함 | LoD 위반 | Hide Delegate |
| 같은 chain 이 여러 service 에 반복 | encapsulation 0 | Hide Delegate + 도메인 method 명명 |
| Hide Delegate 의 과다 적용으로 middle man 폭주 | 두 객체가 거의 같은 method 노출 | Move Method 로 한쪽에 책임 통합 |
| 모든 getter 를 제거하려 함 | DTO / 응답 모델까지 영향 | §8 의 예외 인정 |
| Anemic 객체 + Service 가 모든 결정 | tell-don't-ask 의 정반대 | 도메인 method 로 행위 이동 |

---

## 11. 참고

- [[object-oriented-programming|↑ OOP hub]]
- [[concepts]]
- [[solid-principles]]
- [[good-vs-bad-oop]] §5 case study
- [[anti-patterns]] §4.9 Train Wreck
- [[coupling-cohesion]] §4 결합도
- [[composition-over-inheritance]]
- [[../../../40-patterns/refactoring-patterns/pattern-replace-conditional-with-polymorphism|↗ refactoring 패턴]]
- Andy Hunt, Dave Thomas — "The Pragmatic Programmer" — Tell, Don't Ask
- Karl Lieberherr — "Assuring good style for object-oriented programs" (Northeastern, 1989) — Law of Demeter 원전
- Martin Fowler — "Refactoring" 2판 Hide Delegate / Remove Middle Man
