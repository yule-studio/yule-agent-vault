---
title: "composition over inheritance — 상속의 한계와 컴포지션의 적용"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-18T18:00:00+09:00
tags: [computer-science, oop, composition, inheritance, design]
home_hub: object-oriented-programming
related:
  - "[[object-oriented-programming]]"
  - "[[concepts]]"
  - "[[solid-principles]]"
  - "[[coupling-cohesion]]"
  - "[[anti-patterns]]"
---

# composition over inheritance — 상속의 한계와 컴포지션의 적용

**[[object-oriented-programming|↑ object-oriented-programming]]**

---

## 1. 목적

본 문서는 OOP 의 두 가지 코드 재사용 메커니즘 — 상속 (inheritance) 과 컴포지션 (composition) — 의 차이, 상속의 한계, 컴포지션의 적용 절차를 정의한다.

본 문서가 정의하는 것:
- 상속과 컴포지션의 구조적 차이
- 상속이 적합한 경우 / 부적합한 경우
- 컴포지션으로 전환하는 절차
- GoF 의 "Favor composition over inheritance" 원칙의 근거
- delegation / mixin / trait 등 컴포지션 변형
- Go / Rust 가 상속을 제거하고 composition 만 채택한 이유

본 문서가 정의하지 않는 것:
- LSP 의 형식 정의 — [[solid-principles]] §4
- 4 pillar 의 상속 정의 — [[concepts]] §8

---

## 2. 범위

| 구분 | 포함 |
| --- | --- |
| 대상 | OOP 의 코드 재사용 기법 |
| 포함 | inheritance / composition / delegation / mixin / trait |
| 제외 | 함수 합성 (FP) — 별도 |

---

## 3. 용어

| 용어 | 정의 |
| --- | --- |
| **inheritance (상속)** | 자식 class 가 부모 class 의 구조 / 행위를 물려받는 관계. "is-a". |
| **composition (컴포지션)** | 한 객체가 다른 객체를 field 로 보유하고 그 행위를 호출하는 관계. "has-a". |
| **delegation (위임)** | 자기에게 들어온 메시지를 다른 객체에게 그대로 넘기는 것. composition 의 특수 형태. |
| **mixin** | 부분 구현 + interface 의 결합. 다중 상속의 안전한 형태 (Ruby module / Scala trait). |
| **trait** | Rust / Scala 의 mixin 등가물. method 시그니처 + default impl. |
| **interface inheritance** | 명세만 상속 (구현 0). Java interface / Go interface / Rust trait. |
| **implementation inheritance** | 구현도 상속. 일반적인 `extends`. |
| **fragile base class** | 부모 변경이 자식의 동작을 무작위로 깨뜨리는 현상. |

---

## 4. 구조적 차이

### 4.1 상속

```java
class Account {
    protected long balance;
    void deposit(long amount) { balance += amount; }
    void withdraw(long amount) { balance -= amount; }
}

class SavingsAccount extends Account {       // is-a
    void applyInterest() { balance += (long)(balance * 0.01); }
}
```

| 특성 | 효과 |
| --- | --- |
| 결합 | 컴파일 타임 (compile-time binding) |
| 변경 | 부모 변경이 자식 모두에 영향 |
| 다형성 | 자동 (자식이 부모를 대체 가능) |
| 다중 | 대부분 언어가 single inheritance 만 |
| 캡슐화 | 부모의 protected field 를 자식이 직접 접근 → encapsulation 일부 깨짐 |

### 4.2 컴포지션

```java
class Account {
    private long balance;
    void deposit(long amount) { balance += amount; }
    void withdraw(long amount) { balance -= amount; }
    long balance() { return balance; }
}

class SavingsAccount {                        // has-a
    private final Account account;
    private final double interestRate;
    SavingsAccount(Account a, double r) {
        this.account = a;
        this.interestRate = r;
    }
    void deposit(long amount) { account.deposit(amount); }   // delegation
    void applyInterest() {
        long interest = (long)(account.balance() * interestRate);
        account.deposit(interest);
    }
}
```

| 특성 | 효과 |
| --- | --- |
| 결합 | 런타임 (runtime binding) |
| 변경 | 내부 객체 변경이 외부에 노출되지 않음 |
| 다형성 | 인터페이스 기반 — Account 가 interface 면 다른 구현 주입 가능 |
| 다중 | 자유 (여러 객체 보유) |
| 캡슐화 | 내부 객체의 method 만 통해 접근 |

---

## 5. 상속의 한계

### 5.1 fragile base class

부모 변경이 자식의 동작을 무작위로 깨뜨린다.

```java
class List {
    void add(Object o)      { /* */ }
    void addAll(Collection c) { for (Object o : c) add(o); }   // ← 내부적으로 add 호출
}

class CountingList extends List {
    private int count = 0;
    @Override
    void add(Object o) { count++; super.add(o); }
    @Override
    void addAll(Collection c) { count += c.size(); super.addAll(c); }   // 중복 카운트
}
```

`addAll` 안에서 부모가 `add` 를 호출 → 자식의 오버라이드된 add 가 호출되어 count 가 중복 증가. **부모의 구현 detail 이 자식의 가정을 깨뜨림**.

→ Effective Java (Bloch) Item 18 — "Favor composition over inheritance" 의 핵심 예.

### 5.2 LSP 위반 빈도

분류상의 "is-a" 가 행위상의 "is-a" 와 다른 경우, 자식이 부모를 대체 못함.

| 분류상 is-a | 행위상 is-a 위반 |
| --- | --- |
| Square is-a Rectangle | width/height invariant 다름 |
| Stack is-a List | pop / push 만 허용해야 함 — 중간 insert 노출 X |
| ReadOnlyList is-a List | mutation method 가 던지는 예외 → 사전 조건 강화 |

상세: [[solid-principles]] §4 LSP.

### 5.3 다중 상속의 모호성

```python
class A:    def hello(self): print("A")
class B(A): def hello(self): print("B")
class C(A): def hello(self): print("C")
class D(B, C): pass    # B 와 C 중 어느 hello?

D().hello()    # B (MRO: D → B → C → A)
```

Python 의 MRO (C3 linearization), C++ 의 virtual inheritance 가 해결하지만 복잡도 증가.

### 5.4 깊은 계층의 navigation 비용

`class Animal → Mammal → Carnivore → Felidae → Cat → DomesticCat` 같은 6 단계 계층은 한 method 의 동작을 추적하기 위해 6 파일을 읽어야 한다. 디버깅 비용 ↑.

### 5.5 변경의 cascade

부모의 method signature 변경 → 모든 자식 갱신. SOLID 의 SRP / OCP 와 충돌 발생.

---

## 6. 컴포지션이 적합한 경우 / 부적합한 경우

### 6.1 컴포지션이 적합

| 조건 | 이유 |
| --- | --- |
| "has-a" 관계가 명확 | 객체가 다른 객체를 사용 |
| 다중 행위 결합 (mixin) | 여러 능력을 조합 |
| 런타임 교체 가능성 | 전략 변경 |
| 캡슐화 강화 필요 | 내부 객체 노출 차단 |
| 변경 영향 최소화 | composition 만 변경 |

### 6.2 상속이 여전히 적합

| 조건 | 이유 |
| --- | --- |
| 진정한 "is-a" + 행위 LSP 충족 | 분류상 + 행위상 모두 일치 |
| 부모가 final / sealed / abstract — 의도된 확장점 | 깨지기 쉬운 base class 위험 통제 |
| framework 의 hook (Spring `@Controller` extends X) | 프레임워크가 요구 |
| 인터페이스 상속 (`implements`) | 구현 없는 명세만 — 위험 0 |
| sealed hierarchy + exhaustive pattern match | 컴파일러가 안전성 보장 |

→ Effective Java 의 권고: "design for inheritance or prohibit it" (`final` 또는 `sealed`).

---

## 7. 컴포지션으로 전환하는 절차

### 7.1 전환 절차

| 단계 | 작업 |
| --- | --- |
| 1 | 자식 class 가 사용하는 부모 method 식별 |
| 2 | 부모를 interface 로 추출 (필요시) |
| 3 | 자식 class 에서 `extends` 제거, 부모 객체를 field 로 보유 |
| 4 | 사용하던 method 호출을 delegation 으로 변환 |
| 5 | 부모 method 의 protected field 접근은 새 public method 로 노출 |
| 6 | 테스트로 동작 보존 검증 |

### 7.2 before / after

```java
// before — inheritance
class CountingList extends ArrayList<Object> {
    private int count = 0;
    @Override public boolean add(Object o) { count++; return super.add(o); }
    int count() { return count; }
}

// after — composition + delegation
class CountingList {
    private final List<Object> inner = new ArrayList<>();
    private int count = 0;

    void add(Object o)             { count++; inner.add(o); }
    void addAll(Collection<?> c)   { count += c.size(); inner.addAll(c); }
    Object get(int i)              { return inner.get(i); }
    int size()                     { return inner.size(); }
    int count()                    { return count; }
}
```

→ inner 의 구현 변경 (ArrayList → LinkedList) 에 CountingList 는 영향 없음. fragile base class 도 해소.

---

## 8. 컴포지션 변형

### 8.1 delegation

자기에게 온 메시지를 그대로 다른 객체에게 넘김. boilerplate 많지만 명시적.

```java
class WindowWithLog {
    private final Window window;
    private final Logger log;
    WindowWithLog(Window w, Logger l) { this.window = w; this.log = l; }
    void open()  { log.info("open");  window.open(); }
    void close() { log.info("close"); window.close(); }
    // 모든 method delegate
}
```

→ Kotlin 의 `by` 키워드, Groovy 의 `@Delegate` 가 boilerplate 자동화.

### 8.2 mixin (Ruby / Python)

부분 구현을 가진 module 을 여러 class 에 포함.

```ruby
module Comparable
    def >(other)  = compare(other) > 0
    def <(other)  = compare(other) < 0
    def ==(other) = compare(other) == 0
    # compare 만 구현하면 됨
end

class Version
    include Comparable
    def compare(other) = self.number - other.number
end
```

### 8.3 trait (Scala / Rust)

interface + default 구현. 다중 trait 가능.

```rust
trait Animal {
    fn name(&self) -> String;
    fn introduce(&self) {                    // default impl
        println!("I am {}", self.name());
    }
}

struct Dog;
impl Animal for Dog {
    fn name(&self) -> String { "Dog".into() }
    // introduce 는 default 사용
}
```

### 8.4 strategy pattern

행위를 객체로 추출해 런타임 교체.

```java
interface CompressionStrategy { byte[] compress(byte[] data); }
class GzipCompression implements CompressionStrategy { /* */ }
class ZstdCompression implements CompressionStrategy { /* */ }

class FileStore {
    private final CompressionStrategy compressor;
    FileStore(CompressionStrategy c) { this.compressor = c; }
    void save(byte[] data) { storage.write(compressor.compress(data)); }
}
```

→ GoF Strategy 패턴 = "행위의 컴포지션".

### 8.5 decorator pattern

같은 인터페이스를 구현하면서 다른 객체를 감싸 행위 추가.

```java
interface Coffee { Money cost(); }
class Espresso implements Coffee { public Money cost() { return Money.of(3000); } }
class WithMilk implements Coffee {
    private final Coffee inner;
    WithMilk(Coffee c) { this.inner = c; }
    public Money cost() { return inner.cost().plus(Money.of(500)); }
}
```

→ Java IO 의 `InputStream` 계층이 decorator 의 대표 사용.

---

## 9. Go / Rust 의 선택

### 9.1 Go — 상속 제거

Go 는 명시적으로 상속이 없다. 대신 **struct embedding + interface** 만 제공.

```go
type Animal struct{}
func (a Animal) eat() { /* */ }

type Dog struct {
    Animal       // embedding — composition 의 syntactic shortcut
    name string
}

// d := Dog{}; d.eat()   // Animal 의 eat 가 promoted
```

→ embedding 은 syntactic 컴포지션. Dog 가 Animal 의 method 를 promote 하지만 LSP 같은 substitutability 보장은 interface 가 담당.

### 9.2 Rust — 상속 제거

Rust 도 상속 없음. struct + trait + composition 만.

```rust
struct Logger { /* */ }
struct Server {
    logger: Logger,    // composition
}
trait Handler {
    fn handle(&self, req: Request) -> Response;
}
impl Handler for Server { fn handle(&self, req: Request) -> Response { /* */ } }
```

### 9.3 두 언어가 상속을 제거한 이유

| 이유 | 효과 |
| --- | --- |
| fragile base class 차단 | 깨지기 쉬운 계층 없음 |
| 명시적 의존 | embedded field 가 항상 visible |
| 단순한 type 시스템 | 다중 상속 / MRO 복잡도 없음 |
| 강한 interface 위주 | structural typing (Go) / trait (Rust) 가 다형성 담당 |

→ 두 언어에서 OOP 의 4 pillar 중 inheritance 만 빠지고 나머지 3 (encapsulation / polymorphism / abstraction) 은 그대로.

---

## 10. 적용 결정 절차

새 코드 / 리팩토링 시 다음 순서로 결정.

| 단계 | 질문 | 답 → 결과 |
| --- | --- | --- |
| 1 | 행위 재사용이 필요한가? | 아니오 → 둘 다 불필요 |
| 2 | "is-a" 관계인가, "has-a" 관계인가? | has-a → composition |
| 3 | LSP 충족 가능한가 (자식이 부모 대체 시 사전/사후 조건 보존)? | 아니오 → composition |
| 4 | 부모가 final / sealed / 의도된 확장점인가? | 아니오 → composition |
| 5 | 런타임 교체 가능성이 있는가? | 그렇다 → composition (strategy) |
| 6 | 인터페이스만 상속하는가 (구현 X)? | 그렇다 → 인터페이스 상속 OK |

→ 의심스러우면 composition. 상속은 "정말 모두 충족될 때만".

---

## 11. 흔한 실패 모드

| 실패 | 원인 | 해결 |
| --- | --- | --- |
| 부모 변경이 자식들을 무작위로 깨뜨림 | fragile base class | composition + delegation |
| 깊은 계층 (5+) 의 method 추적 불가 | implementation inheritance 누적 | 평탄화 + composition |
| Square is-a Rectangle 같은 LSP 위반 | 분류상 is-a 와 행위상 is-a 혼동 | composition + 별도 type |
| 다중 상속의 diamond 문제 | 같은 method 의 다중 정의 | interface inheritance + composition |
| boilerplate 폭주 (모든 method delegate) | 명시적 delegation | Kotlin `by` / Lombok `@Delegate` |
| 인터페이스 상속을 implementation 상속으로 착각 | `extends AbstractFoo` 대신 `implements Foo` 가능 | interface 만 의존, 구현 대체 자유 |
| 상속 깊이로 over-engineering | 미리 만든 인터페이스 폭발 | YAGNI + Rule of Three |

---

## 12. 참고

- [[object-oriented-programming|↑ OOP hub]]
- [[concepts]]
- [[solid-principles]] §4 LSP
- [[coupling-cohesion]]
- [[anti-patterns]]
- [[good-vs-bad-oop]]
- [[../design-patterns/strategy/strategy|↗ Strategy pattern]]
- [[../design-patterns/decorator/decorator|↗ Decorator pattern]]
- Joshua Bloch — "Effective Java" Item 18 ("Favor composition over inheritance")
- Gang of Four — "Design Patterns" (1994) — "Favor object composition over class inheritance" (원전)
- Sandi Metz — "Practical Object-Oriented Design" (POODR) — "inheritance is a discount on duplication"
