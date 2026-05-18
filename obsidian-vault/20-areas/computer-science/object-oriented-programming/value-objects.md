---
title: "value objects — immutable / 동등성 / 검증 / 표현력"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-18T20:30:00+09:00
tags: [computer-science, oop, value-object, immutability, ddd]
home_hub: object-oriented-programming
related:
  - "[[object-oriented-programming]]"
  - "[[concepts]]"
  - "[[ddd-tactical-patterns]]"
  - "[[anti-patterns]]"
  - "[[oop-for-backend]]"
---

# value objects — immutable / 동등성 / 검증 / 표현력

**[[object-oriented-programming|↑ object-oriented-programming]]**

---

## 1. 목적

본 문서는 OOP 에서 가장 강력하지만 흔히 누락되는 객체 종류인 **Value Object (VO)** 를 정의하고, primitive obsession 의 해소 / 검증 / 동등성 / 표현력의 측면에서 사용 패턴을 정리한다.

본 문서가 정의하는 것:
- Value Object 의 정의 (identity 없음 + immutable + value equality)
- Entity vs Value Object 의 구분
- Value Object 의 4 가지 책임 (검증 / 표현 / 동작 / 변환)
- 언어별 구현 (Java record / Kotlin data class / Python dataclass / TypeScript / Scala case class)
- 백엔드 도메인의 대표 VO 예 (Money / Email / PhoneNumber / Address / Range / Quantity)
- VO 도입의 효과 (primitive obsession 해소 / 검증 1 곳 집중 / 의미 복구)
- VO 도입의 한계 (성능 / 직렬화 / ORM)

본 문서가 정의하지 않는 것:
- DDD 의 다른 tactical pattern — [[ddd-tactical-patterns]]
- 4 pillar / class — [[concepts]]
- primitive obsession 자체 — [[anti-patterns]] §4.8

---

## 2. 범위

| 구분 | 포함 |
| --- | --- |
| 대상 | OOP 의 immutable 도메인 객체 |
| 적용 | 도메인 모델 / DTO 의 표현력 보강 |
| 제외 | 함수형 언어의 algebraic data type (유사 개념) |
| 제외 | C struct / Go struct — value semantics 만 같음 |

---

## 3. 용어

| 용어 | 정의 |
| --- | --- |
| **Value Object (VO)** | identity 없이 값으로 구분되는 immutable 도메인 객체. |
| **value equality** | 모든 field 가 같으면 동일 객체로 간주. |
| **identity equality** | 메모리 주소 / 식별자가 같아야 동일. |
| **immutable** | 생성 후 상태 변경 불가. 변경은 새 인스턴스 생성. |
| **self-validating** | 생성 시 invariant 를 검증해 invalid state 진입 불가. |
| **side-effect free** | method 호출이 외부 / 자신의 상태를 변경 안 함. |
| **primitive obsession** | 도메인 개념을 primitive type 으로 표현하는 안티패턴. |

---

## 4. Value Object 의 정의

### 4.1 4 속성

| 속성 | 의미 |
| --- | --- |
| **identity 없음** | 같은 값의 두 VO 는 같은 객체로 간주. |
| **immutable** | 생성 후 변경 불가. 변경 = 새 인스턴스. |
| **self-validating** | 생성 시 invariant 검증. |
| **side-effect free** | method 가 새 값을 반환할 뿐 상태 변경 X. |

### 4.2 Entity 와 비교

| 항목 | Entity | Value Object |
| --- | --- | --- |
| 식별 | identity (id) | value |
| 동등성 | id 비교 | 모든 field 비교 |
| 가변성 | mutable (상태 변경 method) | immutable |
| 예 | Order, Customer, User | Money, Email, Address, Range |
| lifetime | 길다 (DB 저장) | 짧음 (즉시 생성/폐기) |

### 4.3 의미

> "내가 어제 사용한 1000 원과 오늘 사용한 1000 원은 같은 값" — Money 는 VO.
> "내가 어제 만든 주문과 오늘 만든 주문은 같은 금액이어도 다른 주문" — Order 는 Entity.

---

## 5. Value Object 의 4 가지 책임

### 5.1 검증 (validation)

생성 시점에 invariant 를 강제. 이후 invalid state 진입 불가.

```java
record Email(String value) {
    private static final Pattern PATTERN = Pattern.compile("^[^@\\s]+@[^@\\s]+\\.[^@\\s]+$");

    Email {                                  // compact constructor
        Objects.requireNonNull(value);
        if (!PATTERN.matcher(value).matches()) {
            throw new IllegalArgumentException("invalid email: " + value);
        }
    }
}

new Email("alice@example.com");   // OK
new Email("invalid");              // throw immediately
```

→ Email type 을 보유한 객체는 절대 invalid email 을 갖지 않는다. 사용처마다 정규식 검증 반복 X.

### 5.2 표현 (expression)

primitive 가 가지지 못한 도메인 의미를 표현.

```java
// 나쁨 — primitive obsession
void transfer(String fromAccount, String toAccount, BigDecimal amount, String currency) { }

// 좋음
void transfer(AccountId from, AccountId to, Money amount) { }
```

→ method 시그니처만으로 의도 명확. 인자 순서 실수 (`fromAccount`/`toAccount` 바뀌어도 컴파일 통과) 도 방지.

### 5.3 동작 (behavior)

VO 가 자기 자신과 관련된 동작을 보유.

```java
record Money(BigDecimal amount, Currency currency) {
    Money {
        if (currency == null) throw new IllegalArgumentException();
        amount = amount.setScale(currency.getDefaultFractionDigits(), RoundingMode.HALF_UP);
    }

    Money plus(Money other) {
        if (!currency.equals(other.currency)) throw new CurrencyMismatchException();
        return new Money(amount.add(other.amount), currency);
    }

    Money minus(Money other) {
        if (!currency.equals(other.currency)) throw new CurrencyMismatchException();
        return new Money(amount.subtract(other.amount), currency);
    }

    Money multiply(BigDecimal rate) {
        return new Money(amount.multiply(rate), currency);
    }

    boolean isGreaterThan(Money other) {
        if (!currency.equals(other.currency)) throw new CurrencyMismatchException();
        return amount.compareTo(other.amount) > 0;
    }
}
```

→ `Money` 산술이 외부 service 에 흩어지지 않음.

### 5.4 변환 (conversion)

VO 간 / VO ↔ primitive 변환.

```java
record Temperature(BigDecimal value, Scale scale) {
    Temperature toCelsius() {
        return switch (scale) {
            case CELSIUS -> this;
            case FAHRENHEIT -> new Temperature(value.subtract(32).multiply(5).divide(9), CELSIUS);
            case KELVIN -> new Temperature(value.subtract(273.15), CELSIUS);
        };
    }
}
```

---

## 6. 언어별 구현

### 6.1 Java 16+ — record

```java
record Email(String value) {
    Email { /* validation */ }
}
```

| 자동 | equals / hashCode / toString / accessor / canonical constructor |
| 제한 | final class (상속 불가), instance field 추가 불가 |

### 6.2 Java < 16 — class + Lombok

```java
@Value             // immutable + equals / hashCode / toString
public class Email {
    String value;
    public Email(String value) { /* validation */ this.value = value; }
}
```

### 6.3 Kotlin — data class

```kotlin
data class Email(val value: String) {
    init {
        require(value.matches(emailPattern)) { "invalid email" }
    }
}
```

→ `copy()` method 자동 생성 — `email.copy(value = "new")` 로 변형.

### 6.4 Python — dataclass(frozen=True)

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Email:
    value: str

    def __post_init__(self) -> None:
        if "@" not in self.value:
            raise ValueError(f"invalid email: {self.value}")
```

### 6.5 TypeScript — readonly + brand type

```typescript
type Email = { readonly __brand: "Email"; readonly value: string };

function makeEmail(value: string): Email {
    if (!value.includes("@")) throw new Error(`invalid email: ${value}`);
    return { __brand: "Email", value };
}
```

→ brand type 이 `string` 과 구분되도록 컴파일러가 강제.

### 6.6 Scala — case class

```scala
case class Email(value: String) {
    require(value.contains("@"), s"invalid email: $value")
}
```

→ pattern matching 가능 (`email match { case Email(v) => ... }`).

---

## 7. 백엔드 도메인의 대표 VO

| VO | primitive 대체 | 검증 / 동작 |
| --- | --- | --- |
| **Money** | `BigDecimal + String` | currency 일치 / 산술 / 비교 |
| **Email** | `String` | 정규식 / 도메인 추출 |
| **PhoneNumber** | `String` | 국가 코드 / 정규화 (libphonenumber) |
| **Address** | `String × N` | 우편번호 / 국가 / 정규화 |
| **CustomerId / OrderId** | `String` 또는 `Long` | UUID / KSUID / 형식 |
| **Range / Period / DateRange** | `Date × 2` | start ≤ end / 겹침 검사 / 포함 검사 |
| **Quantity** | `int` | 0 이상 / 단위 / 변환 (kg/lb/cm/inch) |
| **Coordinate** | `double × 2` | 위도 ≤ 90 / 경도 ≤ 180 / 거리 계산 |
| **Password (hashed)** | `String` | hash 알고리즘 / verify |
| **PercentageRate** | `double` | 0-1 범위 / 비교 / 산술 |
| **URL** | `String` | 형식 / scheme |
| **IpAddress** | `String` | IPv4 / IPv6 형식 + range 검사 |

---

## 8. 도입 효과

### 8.1 primitive obsession 해소

```java
// before
void notifyUser(String userId, String message, double charge, String currency, int retryCount) { }

// after
void notifyUser(UserId userId, NotificationMessage message, Money charge, RetryCount retryCount) { }
```

| 효과 | 결과 |
| --- | --- |
| 인자 순서 실수 차단 | 컴파일 타임 type 안전 |
| 도메인 의미 복원 | method 시그니처가 문서화 |
| 검증 한 곳 | UserId / NotificationMessage / Money / RetryCount 의 self-validation |

### 8.2 검증 1 곳 집중

```java
// before — 모든 사용처에서 검증
void createUser(String email) {
    if (!email.matches(EMAIL_REGEX)) throw new IllegalArgumentException();
    // ...
}
void updateEmail(String email) {
    if (!email.matches(EMAIL_REGEX)) throw new IllegalArgumentException();
    // ...
}

// after — Email VO 안에 한 번만
void createUser(Email email) { /* email 은 이미 valid 보장 */ }
void updateEmail(Email email) { /* */ }
```

### 8.3 의미 보존

```java
// before
if (a == b) {            // String vs String? OrderId vs CustomerId 혼동 가능
    // ...
}

// after
if (orderId.equals(other.orderId)) {   // type 으로 의도 명확
    // ...
}
```

### 8.4 테스트 명확

```java
// before
assertThat(result).isEqualTo("user-123");
// → 이 문자열이 무엇? id? name? error code?

// after
assertThat(result.id()).isEqualTo(new UserId("user-123"));
// → type 으로 의도 명확
```

---

## 9. 도입 한계 / 비용

| 한계 | 대응 |
| --- | --- |
| **객체 할당 비용** | 일반적으로 무시 가능. 핫스팟 측정 후 escape analysis / Valhalla (Java 21+) 기대 |
| **직렬화 / JSON** | Jackson / Gson 의 custom serializer 또는 record 자동 지원 |
| **ORM mapping** | JPA `@Embeddable` (Java), 또는 별도 mapper (data mapper 패턴) |
| **API DTO 분리** | DTO 는 별도 / VO 는 도메인. 변환 method 명시 |
| **boilerplate** | record (Java) / data class (Kotlin) / dataclass (Python) 가 거의 해소 |
| **너무 많은 VO** | 핵심 도메인 개념만 — String 모두를 VO 화할 필요 X |

---

## 10. ORM 통합 — JPA 의 @Embeddable

```java
@Embeddable
record Money(BigDecimal amount, String currency) { }

@Entity
class Order {
    @Id private OrderId id;

    @AttributeOverride(name = "amount", column = @Column(name = "total_amount"))
    @AttributeOverride(name = "currency", column = @Column(name = "total_currency"))
    private Money totalPrice;
}
```

→ `total_amount`, `total_currency` 컬럼 2 개로 저장. VO 가 ORM 의 first-class.

---

## 11. 적용 패턴 — VO 점진적 도입 절차

| 단계 | 작업 |
| --- | --- |
| 1 | primitive obsession 신호 식별 (반복 검증, 의미 모호한 String/int) |
| 2 | VO 후보 선정 — 도메인 개념 우선 (Money / Email / PhoneNumber 등) |
| 3 | VO record / class 작성 + 검증 |
| 4 | 한 사용처 (예: 1 endpoint) 만 VO 로 교체 + 테스트 |
| 5 | 점진적 사용처 확대 |
| 6 | 마지막에 ORM mapping 적용 |

→ 도메인 전체를 한 번에 바꾸지 않는다. **새 기능부터 VO** + **변경 시 점진 교체**.

---

## 12. 흔한 실패 모드

| 실패 | 원인 | 정정 |
| --- | --- | --- |
| VO 안에 setter 추가 | mutability 도입 | VO 는 immutable. 변경 = 새 인스턴스 |
| 같은 VO 가 여러 곳에서 다른 검증 | 검증이 외부에 산재 | 검증을 VO constructor 로 통합 |
| `Money.plus` 에 currency 검증 없음 | 잘못된 산술 가능 | currency 다르면 throw |
| VO 가 너무 비대 (10+ method) | Entity 와 혼동 | 동작이 자기 자신 + value 산술인지 확인 |
| Lombok `@Data` 사용 (setter 자동) | mutability + equals 모두 깨질 수 있음 | `@Value` 사용 (immutable) |
| toString 에 PII 포함 | 로그 유출 | 민감 필드는 마스킹 |
| equals/hashCode 일관성 깨짐 | 수동 작성 시 실수 | record / data class / @Value 사용 |
| record 에 비즈니스 로직 너무 많이 | 표현 + 검증을 넘어 흐름 조율 | Entity / Service 로 이동 |

---

## 13. 참고

- [[object-oriented-programming|↑ OOP hub]]
- [[concepts]]
- [[ddd-tactical-patterns]] — Aggregate / Entity 와의 위치
- [[anti-patterns]] §4.8 Primitive Obsession
- [[oop-for-backend]] §8 Value Object 구분
- Eric Evans — "Domain-Driven Design" (2003) — Value Object 원전
- Vaughn Vernon — "Implementing Domain-Driven Design" (2013) §6
- Effective Java (Bloch) Item 17 — Minimize mutability
- Java Records — JEP 395 (Java 16) / Pattern Matching
- libphonenumber — https://github.com/google/libphonenumber
