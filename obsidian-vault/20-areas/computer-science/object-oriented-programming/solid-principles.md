---
title: "SOLID 원칙 — SRP / OCP / LSP / ISP / DIP"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-18T17:00:00+09:00
tags: [computer-science, oop, solid, principles]
home_hub: object-oriented-programming
related:
  - "[[object-oriented-programming]]"
  - "[[concepts]]"
  - "[[coupling-cohesion]]"
  - "[[composition-over-inheritance]]"
  - "[[good-vs-bad-oop]]"
---

# SOLID 원칙 — SRP / OCP / LSP / ISP / DIP

**[[object-oriented-programming|↑ object-oriented-programming]]**

---

## 1. 목적

본 문서는 Robert C. Martin 이 정리한 OOP 의 5 원칙 (SOLID) 의 각 원칙에 대해 정의 / 위반 신호 / 적용 / 코드 비교를 제공한다.

본 문서가 정의하는 것:
- 각 원칙의 정의 (Uncle Bob 의 1995 글 + Clean Architecture 2017 정리 기준)
- 원칙 위반의 객관적 신호
- 위반 → 적용 리팩토링 방향
- before / after 코드 예 (Java 기준, 일부 Python / TypeScript)

본 문서가 정의하지 않는 것:
- 4 pillar — [[concepts]]
- 결합 / 응집의 종류 — [[coupling-cohesion]]
- 상속 vs 컴포지션 — [[composition-over-inheritance]]

---

## 2. SRP — Single Responsibility Principle

### 2.1 정의

> "한 클래스는 변경의 이유가 하나여야 한다."

원래 Uncle Bob 의 표현: "A class should have only one reason to change". Clean Architecture (2017) 에서 "responsibility = 변경을 요청하는 사람 (actor)" 으로 정정.

### 2.2 위반 신호

| 신호 | 의미 |
| --- | --- |
| 한 클래스가 여러 actor (재무팀 / 마케팅 / 운영) 의 요구를 동시에 충족 | 책임 혼재 |
| 클래스 이름이 "Manager" / "Helper" / "Utils" / "Processor" | 모호한 책임 |
| 한 클래스의 method 가 서로 다른 데이터를 다룸 (low cohesion) | [[coupling-cohesion]] §4 |
| 한 PR 이 동일 클래스의 무관한 method 2 개 이상 수정 | shotgun surgery 의 반대 — 한 곳에 모든 변경 |

### 2.3 위반 — before

```java
class Report {
    String calculateTotal(Order o)    { /* 재무팀 요구 */ }
    String renderPdf(Order o)         { /* UI 팀 요구 */ }
    void   sendEmail(Order o)         { /* 운영팀 요구 */ }
    void   logToFile(Order o)         { /* DevOps 요구 */ }
}
```

→ 4 actor 의 요구 변경이 모두 같은 클래스를 건드림.

### 2.4 적용 — after

```java
class ReportCalculator { String calculateTotal(Order o)   { /* */ } }
class ReportRenderer   { String renderPdf(Report r)        { /* */ } }
class ReportSender     { void   sendEmail(Report r)        { /* */ } }
class ReportLogger     { void   log(Report r)              { /* */ } }
```

### 2.5 함정 — 과도한 분할

method 1 개씩 class 1 개로 쪼개면 navigation 비용 증가 + cohesion 저하. 같은 actor 의 같은 데이터를 다루는 method 는 한 곳에 둔다.

---

## 3. OCP — Open/Closed Principle

### 3.1 정의

> "확장에는 열려 있고, 수정에는 닫혀 있어야 한다."

Bertrand Meyer (1988) 의 원전. 새 기능 추가 시 기존 코드를 수정하지 않고 새 코드를 추가하는 방식으로 확장한다.

### 3.2 위반 신호

| 신호 | 의미 |
| --- | --- |
| 새 type 추가마다 if/else / switch 분기 추가 | type 별 동작이 한 함수 안에 모임 |
| `instanceof` 분기 | polymorphism 무력화 |
| 새 결제 수단 추가 시 결제 모듈의 핵심 파일을 수정 | 변경의 cascade |
| enum 추가마다 모든 switch 갱신 | exhaustive switch 부재 시 위험 |

### 3.3 위반 — before

```java
class PaymentProcessor {
    void process(String type, long amount) {
        if (type.equals("card")) { /* 카드 처리 */ }
        else if (type.equals("bank")) { /* 계좌이체 처리 */ }
        else if (type.equals("paypal")) { /* PayPal 처리 */ }
        // 새 결제 수단마다 여기 수정
    }
}
```

### 3.4 적용 — after

```java
interface Payment { void process(long amount); }

class CardPayment implements Payment { public void process(long a) { /* */ } }
class BankPayment implements Payment { public void process(long a) { /* */ } }
class PaypalPayment implements Payment { public void process(long a) { /* */ } }

class PaymentProcessor {
    void process(Payment p, long amount) { p.process(amount); }
    // 새 결제 수단은 Payment 구현 추가 — PaymentProcessor 변경 0
}
```

### 3.5 함정 — 과도한 추상화

미래의 모든 확장을 예측해서 인터페이스 폭발을 만들면 YAGNI 위반. **두 번째 변형이 생긴 시점에 인터페이스 도입** (Rule of Three) 이 안전.

---

## 4. LSP — Liskov Substitution Principle

### 4.1 정의

> "subtype 의 object 는 supertype 의 object 로 대체 가능해야 한다."

Barbara Liskov (1987). 자식이 부모를 대체했을 때 호출자의 코드는 부모를 가정하고 작성되어 있어도 정상 동작해야 한다.

### 4.2 LSP 의 정확한 조건

| 조건 | 의미 |
| --- | --- |
| **사전 조건 강화 금지** | 자식이 부모보다 더 까다로운 입력을 요구하면 안 됨 |
| **사후 조건 약화 금지** | 자식이 부모보다 더 약한 결과 보장이면 안 됨 |
| **invariant 유지** | 부모의 불변식을 자식이 깨면 안 됨 |
| **예외 type 보존** | 자식이 부모가 던지지 않는 예외를 던지면 안 됨 |

### 4.3 고전 예 — 정사각형 / 직사각형

```java
class Rectangle {
    protected int w, h;
    void setWidth(int w)  { this.w = w; }
    void setHeight(int h) { this.h = h; }
    int area()            { return w * h; }
}

class Square extends Rectangle {
    @Override
    void setWidth(int w)  { this.w = w; this.h = w; }  // invariant: w == h
    @Override
    void setHeight(int h) { this.w = h; this.h = h; }
}

// 호출자
void test(Rectangle r) {
    r.setWidth(5);
    r.setHeight(10);
    assert r.area() == 50;     // Rectangle 가정. Square 면 100 → 깨짐.
}
```

→ Square 는 Rectangle 을 대체할 수 없다. 분류상의 "is-a" 가 행위 상의 "is-a" 와 다르다.

### 4.4 위반의 결과 — 깨지기 쉬운 base class

[[concepts]] §8.4 의 FeeAccount 예 참고. 자식이 부모의 동작을 다르게 만들면 모든 호출자가 자식 type 을 알아야 한다.

### 4.5 적용

- 자식에서 부모의 행위를 약화 / 강화하지 않는다.
- 정말 행위가 다르면 상속이 아닌 별도 type + composition.
- Design by Contract (Eiffel / Meyer) 의 사전/사후 조건을 명시.

상세: [[composition-over-inheritance]].

---

## 5. ISP — Interface Segregation Principle

### 5.1 정의

> "클라이언트는 자신이 사용하지 않는 method 에 의존하지 않아야 한다."

큰 interface 하나보다 작은 interface 여러 개가 낫다.

### 5.2 위반 신호

| 신호 | 의미 |
| --- | --- |
| 인터페이스의 method 절반을 빈 구현 (`throw NotImplementedException`) 으로 채움 | 사용 안 하는 method 강제 의존 |
| Mock 작성 시 method 수가 너무 많아 부담 | interface 가 너무 큼 |
| 한 interface 의 method 가 서로 다른 client 가 사용 | client 별로 분리 가능 |

### 5.3 위반 — before

```java
interface Worker {
    void work();
    void eat();          // 로봇 worker 는 안 먹음
    void sleep();        // 로봇 worker 는 안 잠
}

class Human implements Worker { /* 다 구현 */ }

class Robot implements Worker {
    public void work() { /* */ }
    public void eat()  { throw new UnsupportedOperationException(); }
    public void sleep(){ throw new UnsupportedOperationException(); }
}
```

### 5.4 적용 — after

```java
interface Workable { void work(); }
interface Feedable { void eat(); }
interface Sleepable { void sleep(); }

class Human implements Workable, Feedable, Sleepable { /* */ }
class Robot implements Workable { /* */ }
```

### 5.5 함정 — 인터페이스 폭발

극단으로 가면 method 1 개당 interface 1 개 — 응집을 깨뜨림. **client 가 묶어서 사용하는 method 들은 같은 interface**.

---

## 6. DIP — Dependency Inversion Principle

### 6.1 정의

> "고수준 모듈은 저수준 모듈에 의존하면 안 된다. 둘 다 추상에 의존해야 한다."
> "추상은 구체에 의존하면 안 된다. 구체가 추상에 의존해야 한다."

### 6.2 위반 신호

| 신호 | 의미 |
| --- | --- |
| 비즈니스 로직 (use case) 이 ORM / HTTP client / 파일 시스템을 직접 `new` | 저수준 의존 |
| 테스트 시 DB / 외부 API 가 필요 | 추상 부재 |
| 비즈니스 로직 변경 없이 인프라 (PostgreSQL → MySQL) 교체 불가 | 의존 방향 잘못 |

### 6.3 위반 — before

```java
class OrderService {
    private MySqlOrderRepository repo = new MySqlOrderRepository();    // 고수준 → 저수준 직접

    void place(Order o) {
        repo.save(o);
        new SmtpEmailSender().send(o.getCustomerEmail(), "주문 완료");
    }
}
```

→ OrderService 가 MySQL 과 SMTP 에 직결. 테스트 불가, 교체 불가.

### 6.4 적용 — after

```java
// 추상 (고수준 모듈 이 선언)
interface OrderRepository { void save(Order o); }
interface EmailSender { void send(String to, String body); }

// 고수준 모듈 — 추상에만 의존
class OrderService {
    private final OrderRepository repo;
    private final EmailSender sender;
    OrderService(OrderRepository r, EmailSender s) { this.repo = r; this.sender = s; }
    void place(Order o) {
        repo.save(o);
        sender.send(o.getCustomerEmail(), "주문 완료");
    }
}

// 저수준 모듈 — 추상을 구현
class MySqlOrderRepository implements OrderRepository { /* */ }
class SmtpEmailSender implements EmailSender { /* */ }
```

### 6.5 의존성 주입 (DI) 과의 관계

DIP 는 원칙, DI 는 그 원칙을 구현하는 기법 중 하나. DI 가 자동 (Spring / Guice) 이든 수동 (constructor 주입) 이든, DIP 충족 여부와는 별개.

| 항목 | 정의 |
| --- | --- |
| DIP | 의존 방향이 추상으로 향해야 한다 (원칙) |
| DI | 의존성을 외부에서 주입하는 기법 (생성자 / setter / factory) |
| IoC | 제어 흐름을 framework 에 위임 (큰 개념) |
| DI container | 의존 그래프를 자동 wiring 하는 도구 (Spring / Guice / Dagger) |

→ DI 사용해도 DIP 위반 가능 (구체 타입을 주입하면). DIP 핵심은 **누가 인터페이스를 정의하는가** — 고수준 모듈이 정의해야 한다.

---

## 7. SOLID 원칙 간 상호작용

| 관계 | 효과 |
| --- | --- |
| SRP + OCP | 책임이 분리되면 한 책임의 확장이 다른 책임을 건드리지 않음 |
| OCP + LSP | OCP 가 polymorphism 으로 실현 — LSP 가 깨지면 OCP 도 깨짐 |
| ISP + DIP | 작은 interface 가 더 정확한 의존 선언 가능 |
| SRP + ISP | 같은 actor 의 method 끼리 묶이면 자연스럽게 ISP 충족 |
| LSP + composition | LSP 가 어려우면 상속 대신 composition 으로 우회 |

→ 5 원칙은 독립이 아니라 **결합 / 응집의 다양한 측면**. [[coupling-cohesion]] 이 통합 관점.

---

## 8. 실패 모드 — SOLID 의 오용

| 실패 | 원인 | 정정 |
| --- | --- | --- |
| 모든 method 마다 인터페이스 생성 | ISP / DIP 의 극단 적용 | 두 번째 구현이 필요한 시점에 도입 |
| 추상 클래스만 잔뜩 + 구체 거의 없음 | 미래 가정 | YAGNI 적용 |
| Service / Manager / Helper / Util 4 단어로 모든 class | SRP 무시 + 책임 모호 | 도메인 용어로 명명 |
| 같은 PR 이 10 개 파일을 동시 수정 | SRP 또는 OCP 위반 | 책임 / 확장점 재설계 |
| 모든 의존성을 인터페이스로 감싸고 1 구현만 존재 | DIP 형식만 적용 | 두 번째 구현 가능성이 없으면 직접 의존 OK |
| Liskov 위반 (Square / Rectangle 패턴) | 분류상 is-a 와 행위상 is-a 혼동 | 상속 대신 composition |
| 큰 god interface | ISP 위반 | client 별로 분리 |
| Mock 이 너무 복잡 | interface 가 너무 큼 (ISP) 또는 의존이 너무 많음 (SRP) | 책임 재분배 |

---

## 9. SOLID 의 한계

| 한계 | 보완 |
| --- | --- |
| OOP 한정 — FP / Actor 모델엔 부분만 적용 | 함수형 원칙 (immutability / pure function) 도 같이 학습 |
| 객체 간 협력 패턴은 별도 — GoF | [[../design-patterns/design-patterns]] |
| 분산 시스템의 결합 / 일관성은 별도 | [[../distributed-systems/distributed-systems]] |
| 측정 도구 부족 — 정성 평가 | [[coupling-cohesion]] 의 정량 지표 + 코드 메트릭 (Cyclomatic / Cognitive) |

---

## 10. 참고

- [[object-oriented-programming|↑ OOP hub]]
- [[concepts]]
- [[coupling-cohesion]]
- [[composition-over-inheritance]]
- [[anti-patterns]]
- [[good-vs-bad-oop]]
- [[../design-patterns/design-patterns]]
- Robert C. Martin — "Clean Architecture" (2017) §7-11
- Robert C. Martin — "The Principles of OOD" (2000, original blog)
- Barbara Liskov, Jeannette Wing — "A Behavioral Notion of Subtyping" (1994)
- Bertrand Meyer — "Object-Oriented Software Construction" (1997) — Open/Closed 원전
