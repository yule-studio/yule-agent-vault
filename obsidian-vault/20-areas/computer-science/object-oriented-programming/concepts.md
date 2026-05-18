---
title: "OOP 핵심 개념 — 4 pillar / class / object / message / interface"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-18T16:30:00+09:00
tags: [computer-science, oop, concepts, encapsulation, inheritance, polymorphism, abstraction]
home_hub: object-oriented-programming
related:
  - "[[object-oriented-programming]]"
  - "[[solid-principles]]"
  - "[[composition-over-inheritance]]"
  - "[[coupling-cohesion]]"
---

# OOP 핵심 개념 — 4 pillar / class / object / message / interface

**[[object-oriented-programming|↑ object-oriented-programming]]**

---

## 1. 목적

본 문서는 OOP 의 기본 용어와 4 pillar (encapsulation / inheritance / polymorphism / abstraction) 를 정의한다.

본 문서가 정의하는 것:
- object / class / message / interface 의 정의와 관계
- 4 pillar 각각의 정의 / 동작 / 적용 / 위반 시 결과
- OOP 와 인접 패러다임 (구조적 / 함수형) 의 차이
- 언어별 OOP 구현 (Java / C++ / Python / JavaScript / Ruby / Go / Rust) 의 특징

본 문서가 정의하지 않는 것:
- SOLID 원칙 — [[solid-principles]]
- 결합 / 응집의 종류 — [[coupling-cohesion]]
- 안티패턴 — [[anti-patterns]]

---

## 2. 범위

| 구분 | 포함 |
| --- | --- |
| 대상 | 일반 OOP 개념 (언어 중립) + 주요 언어별 특징 |
| 제외 | 특정 프레임워크 (Spring / Django) — [[oop-for-backend]] |
| 제외 | DDD / Aggregate / Bounded Context — [[../software-engineering/software-engineering]] §5.4 |

---

## 3. 용어

| 용어 | 정의 |
| --- | --- |
| **object (객체)** | 상태 (state) + 행위 (behavior) + 식별 (identity) 을 갖는 실행 단위. |
| **state** | object 가 보유한 데이터의 현재 값. |
| **behavior** | object 가 응답하는 메시지의 집합. |
| **identity** | object 를 다른 동일 상태의 object 와 구분하는 고유성. 보통 메모리 주소 / UUID. |
| **class** | 같은 종류 object 의 구조 / 동작을 정의하는 template. 일부 언어 (JavaScript ~ES5, Self) 는 class 없이 prototype 만 사용. |
| **instance** | class 로부터 생성된 구체 object. |
| **method** | object 가 수신할 수 있는 메시지에 대한 처리 코드. |
| **message** | object 에게 어떤 일을 요청하는 호출. Smalltalk 식 표현에서는 method 호출 = 메시지 전송. |
| **field / attribute / property** | object 의 상태를 저장하는 변수. |
| **constructor** | instance 생성 시 호출되는 초기화 method. |
| **interface** | object 가 응답해야 하는 메시지 집합의 명세 (구현 없음). |
| **abstract class** | 일부 구현을 갖되 직접 instance 화 불가능한 class. |
| **type** | object 가 응답 가능한 메시지의 집합. 정적 type 시스템 (Java) 은 컴파일 시점에 검사. |

---

## 4. object 의 정의 — state + behavior + identity

### 4.1 3 요소

| 요소 | 예 (은행 계좌) |
| --- | --- |
| state | balance: 50000, owner: "user-1" |
| behavior | deposit(amount), withdraw(amount), getBalance() |
| identity | account-id "ACC-9f3a" (같은 balance 의 다른 계좌와 구분) |

### 4.2 동일성 vs 동등성

| 비교 | 의미 |
| --- | --- |
| **identity equality** (`==` in Java, `is` in Python) | 같은 object 인가 (같은 주소) |
| **value equality** (`.equals()`, `__eq__`) | 같은 값을 가지는가 |

→ 두 사람이 같은 balance 를 가져도 계좌는 다르다 (identity ≠).

### 4.3 lifetime

| 단계 | 동작 |
| --- | --- |
| 생성 | constructor 호출 |
| 사용 | message 수신 → method 실행 → state 변경 |
| 소멸 | 명시적 (`destructor`) 또는 GC |

---

## 5. class 와 instance

### 5.1 정의

class 는 instance 의 구조 (field) 와 동작 (method) 을 정의한 template.

```java
// Java
class Account {
    private long balance;       // field
    private final String id;    // immutable identity

    Account(String id, long initial) {
        this.id = id;
        this.balance = initial;
    }

    void deposit(long amount) { balance += amount; }
    long getBalance()         { return balance; }
}

Account a = new Account("ACC-1", 50000);   // instance 생성
```

### 5.2 class 의 책임

| 책임 | 효과 |
| --- | --- |
| 구조 정의 | field 종류 / 타입 |
| 동작 정의 | method 시그니처와 구현 |
| 생성 통제 | constructor 의 가시성 + 검증 |
| 불변식 보장 | constructor + setter 검증으로 invalid state 차단 |

### 5.3 prototype 기반 (JavaScript / Self)

class 가 없는 언어는 object 가 다른 object 를 prototype 으로 참조한다.

```js
const account = { balance: 50000 };
const myAccount = Object.create(account);
myAccount.balance = 100;       // 자기 자신에 새 field
```

→ ES6 `class` 는 prototype 의 syntactic sugar.

---

## 6. message — 객체 간 통신

### 6.1 정의

OOP 의 원형 (Smalltalk) 에서 method 호출은 "object 에게 메시지를 보낸다" 로 표현된다. object 는 메시지를 받고 응답할지 (이 method 가 있으면) 또는 거부할지 (없으면 errorMessage) 결정한다.

```smalltalk
account deposit: 1000.       " account 에게 deposit: 메시지 + 인자 1000 "
```

### 6.2 OOP 의 통신 원칙

| 원칙 | 정의 |
| --- | --- |
| object 는 자기 state 를 외부에 노출하지 않는다 | encapsulation |
| 다른 object 의 state 를 직접 만지지 않는다 | tell, don't ask |
| 다른 object 의 친구의 친구의 state 를 만지지 않는다 | Law of Demeter |
| 같은 메시지에 다른 object 가 다르게 응답할 수 있다 | polymorphism |

---

## 7. 4 pillar 1 — encapsulation (캡슐화)

### 7.1 정의

object 의 state 를 외부에서 직접 접근할 수 없게 감추고, 변경은 method 를 통해서만 가능하게 하는 원칙.

### 7.2 적용

```java
class Account {
    private long balance;        // 외부 접근 차단

    void deposit(long amount) {
        if (amount <= 0) throw new IllegalArgumentException();
        balance += amount;
    }

    void withdraw(long amount) {
        if (amount > balance) throw new InsufficientBalanceException();
        balance -= amount;
    }
}
```

### 7.3 효과

| 효과 | 결과 |
| --- | --- |
| **invariant 보장** | balance 가 0 이하로 가지 않음을 method 안에서 강제 |
| **구현 교체 자유** | balance 를 long → BigDecimal 로 바꿔도 외부 영향 0 |
| **변경 추적** | balance 가 바뀐 시점을 method 안에서만 찾으면 됨 |
| **잘못된 사용 방어** | invalid state 전이를 원천 차단 |

### 7.4 위반 — anemic 객체

```java
// 나쁨 — getter / setter 만 있고 행위 없음
class Account {
    private long balance;
    public long getBalance()        { return balance; }
    public void setBalance(long b)  { this.balance = b; }     // 외부가 임의 변경
}

// 외부 코드가 invariant 책임
account.setBalance(account.getBalance() - 1000);   // 마이너스 가능
```

→ encapsulation 위반의 가장 흔한 형태. 상세: [[anti-patterns]] (anemic domain model).

### 7.5 가시성 modifier

| Java | Python | TypeScript | 의미 |
| --- | --- | --- | --- |
| public | (default) | public | 외부 접근 가능 |
| protected | `_name` (convention) | protected | subclass / same package |
| private | `__name` (name mangling) | private | 같은 class 만 |
| (package-private) | - | - | 같은 package |

→ Python / TypeScript 는 컴파일러가 강제하지 않는 convention. 그래도 동일 의미로 사용.

---

## 8. 4 pillar 2 — inheritance (상속)

### 8.1 정의

기존 class (parent / superclass) 의 구조와 동작을 새 class (child / subclass) 가 물려받는 메커니즘.

```java
class Account {
    protected long balance;
    void deposit(long amount) { balance += amount; }
}

class SavingsAccount extends Account {
    private double interestRate;
    void applyInterest() { balance += (long)(balance * interestRate); }
}
```

### 8.2 종류

| 종류 | 의미 |
| --- | --- |
| **single inheritance** | parent 1 개 (Java / C# / Ruby) |
| **multiple inheritance** | parent N 개 (C++ / Python / Eiffel) |
| **interface inheritance** | 명세만 상속 (Java interface, Go interface, Rust trait) |
| **mixin** | 부분 구현 + interface (Ruby module, Python multiple inheritance, Scala trait) |

### 8.3 substitutability (Liskov)

자식은 부모를 대체할 수 있어야 한다. 자식이 부모의 메시지에 다르게 응답하거나 사전 / 사후 조건을 깨면 LSP 위반.

상세: [[solid-principles]] §3 LSP.

### 8.4 위반의 결과 — 깨지기 쉬운 base class 문제

parent 의 변경이 모든 자식의 동작을 깨뜨릴 수 있다.

```java
class Account {
    void transfer(Account to, long amount) {
        this.withdraw(amount);    // (A)
        to.deposit(amount);       // (B)
    }
}

class FeeAccount extends Account {
    @Override
    void withdraw(long amount) {
        super.withdraw(amount + 100);    // 수수료 100
    }
}
// FeeAccount → FeeAccount transfer 시 (A) 가 200 차감, (B) 는 amount 입금 → 100 손실
```

→ 상속의 한계와 대안: [[composition-over-inheritance]].

### 8.5 다이아몬드 문제 (multiple inheritance)

두 parent 가 같은 method 를 갖는 경우 어느 것을 호출할지 모호. C++ 의 virtual inheritance, Python 의 MRO (C3 linearization) 가 해결.

---

## 9. 4 pillar 3 — polymorphism (다형성)

### 9.1 정의

같은 메시지를 다른 type 의 object 가 다르게 처리하는 능력.

### 9.2 종류

| 종류 | 메커니즘 | 예 |
| --- | --- | --- |
| **subtype polymorphism (dynamic dispatch)** | runtime 에 object 의 실제 type 으로 method 선택 | `Animal a = new Dog(); a.sound();` |
| **parametric polymorphism (generics)** | type 을 parameter 로 받는 함수 / class | `List<Account>`, `Optional<T>` |
| **ad-hoc polymorphism (overloading)** | 같은 이름의 method 를 다른 인자 타입으로 정의 | `print(int)`, `print(String)` |
| **coercion (자동 변환)** | type 변환으로 같은 함수 적용 | `int → double` 자동 |

### 9.3 dynamic dispatch 의 의미

```java
interface Payment {
    void process(long amount);
}

class CreditCardPayment implements Payment { void process(long a) { /* ... */ } }
class BankTransferPayment implements Payment { void process(long a) { /* ... */ } }

void checkout(Payment p, long amount) {
    p.process(amount);    // 어떤 구현이 실행될지는 runtime 의 p 의 실제 type
}
```

→ 호출자 `checkout` 은 구체 타입을 모르고도 동작. 새 결제 수단 추가 시 `checkout` 변경 0.

### 9.4 위반 — instanceof 분기

```java
// 나쁨
void process(Payment p, long amount) {
    if (p instanceof CreditCardPayment) {
        ((CreditCardPayment) p).processCard(amount);
    } else if (p instanceof BankTransferPayment) {
        ((BankTransferPayment) p).processTransfer(amount);
    }
}
```

→ polymorphism 을 무효화. 새 결제 수단마다 함수 변경. OCP 위반 ([[solid-principles]] §2).

### 9.5 generic 의 type 안정성

```java
List<String> list = new ArrayList<>();
list.add("hello");          // OK
list.add(1);                // compile error
String s = list.get(0);     // 캐스팅 불필요
```

→ runtime 의 ClassCastException 을 컴파일 타임에 차단.

---

## 10. 4 pillar 4 — abstraction (추상화)

### 10.1 정의

복잡한 구현 세부를 감추고 필수 인터페이스만 노출하는 원칙.

### 10.2 적용

```java
interface PaymentGateway {
    PaymentResult charge(Money amount, Card card);
}
// 사용자는 어떤 외부 PG (Stripe / Toss / KCP) 인지 모르고 호출
```

### 10.3 추상화 = 인터페이스 + 구현 분리

| 추상화 결과 | 효과 |
| --- | --- |
| 인터페이스 선언 | 호출자가 의존하는 것 |
| 구현 (impl) | 인터페이스의 약속을 충족 |
| 호출자는 인터페이스만 알면 됨 | 구현 교체 자유 (DIP) |

→ 상세: [[solid-principles]] §5 DIP.

### 10.4 추상화의 "수준"

| 수준 | 예 |
| --- | --- |
| 너무 낮음 | `void writeBytesToTcpSocket(byte[] b)` — 호출자가 socket 알아야 |
| 적절 | `void sendOrderConfirmation(Order o)` — 도메인 의도 표현 |
| 너무 높음 | `void doStuff(Object o)` — 표현력 0 |

→ "한 함수는 한 추상 수준만" (Clean Code).

---

## 11. OOP 와 다른 패러다임

| 패러다임 | 단위 | 통신 | 상태 |
| --- | --- | --- | --- |
| 구조적 (절차적) | 함수 + 자료 분리 | 함수 호출 | 전역 / 함수 인자 |
| **OOP** | object (상태 + 행위) | message / method | object 안에 캡슐화 |
| 함수형 (FP) | 순수 함수 + immutable | 함수 합성 | immutable 또는 monad |
| Actor 모델 | actor (상태 + 메일박스) | message (비동기) | actor 안 |

→ OOP 의 message-passing 은 Actor 모델의 원형. Erlang / Elixir / Akka 가 대규모 동시성에서 채택.

→ FP 와 OOP 는 배타적이지 않음. Scala / Kotlin / 모던 Java 가 둘을 결합.

---

## 12. 언어별 OOP 의 특징

| 언어 | 특징 |
| --- | --- |
| **Smalltalk** | OOP 의 원형. 모든 것이 object, 모든 호출이 message. 동적 type. |
| **Java** | class 기반, single inheritance + interface, JVM 의 dynamic dispatch. 정적 type. |
| **C++** | multiple inheritance + virtual table. 직접 메모리 제어. |
| **Python** | class 기반, multiple inheritance, duck typing. 모든 attribute mutable. |
| **JavaScript** | prototype 기반 (ES6 class 는 syntactic sugar). 동적 type. |
| **Ruby** | Smalltalk 영향. 모든 것이 object, mixin (module). |
| **C#** | Java 유사 + property / event / LINQ. CLR 의 generics 는 runtime type 보존. |
| **Go** | class 없음. struct + interface (structural typing). 상속 없음, composition 만. |
| **Rust** | class 없음. struct + trait (Haskell type class 영향). 소유권 (ownership) 으로 mutable state 통제. |
| **Kotlin / Scala** | OOP + FP 결합. data class / sealed class / pattern match. |

→ Go / Rust 는 명시적으로 inheritance 를 제거하고 composition + interface 만 제공. [[composition-over-inheritance]] §6 참고.

---

## 13. 흔한 오해

| 오해 | 정정 |
| --- | --- |
| "class = OOP" | class 없는 OOP (JavaScript prototype, Go) 도 가능. 핵심은 object + message. |
| "getter/setter = encapsulation" | getter/setter 만 있으면 anemic. encapsulation = state + 그것을 다루는 행위가 한 객체. |
| "상속 = OOP 의 핵심" | 상속은 4 pillar 중 하나일 뿐, 오히려 남용 시 결합 증가. composition 이 더 유연. |
| "interface 가 많을수록 OOP" | 과도한 추상화는 YAGNI 위반. 필요한 시점에 도입. |
| "OOP 는 느림" | dynamic dispatch overhead 는 JIT / inline 으로 최소화. 실측 전에 가정 금지. |
| "OOP 와 FP 는 배타적" | Scala / Kotlin / 모던 Java 가 둘을 결합. 단위 (object) 와 변환 (function) 은 보완. |

---

## 14. 참고

- [[object-oriented-programming|↑ OOP hub]]
- [[solid-principles]]
- [[composition-over-inheritance]]
- [[coupling-cohesion]]
- [[anti-patterns]]
- [[good-vs-bad-oop]]
- [[oop-for-backend]]
- [[../design-patterns/design-patterns]]
- Alan Kay (1997) — "I made up the term object-oriented and I can tell you I did not have C++ in mind"
- Smalltalk-80: The Language and its Implementation (Goldberg / Robson 1983)
- Bertrand Meyer — Object-Oriented Software Construction (1997)
