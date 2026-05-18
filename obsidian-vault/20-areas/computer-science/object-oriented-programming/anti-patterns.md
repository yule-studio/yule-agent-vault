---
title: "OOP anti-patterns — 카탈로그 / 리팩토링 방향"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-18T19:00:00+09:00
tags: [computer-science, oop, anti-patterns, refactoring]
home_hub: object-oriented-programming
related:
  - "[[object-oriented-programming]]"
  - "[[concepts]]"
  - "[[solid-principles]]"
  - "[[coupling-cohesion]]"
  - "[[good-vs-bad-oop]]"
  - "[[composition-over-inheritance]]"
---

# OOP anti-patterns — 카탈로그 / 리팩토링 방향

**[[object-oriented-programming|↑ object-oriented-programming]]**

---

## 1. 목적

본 문서는 OOP 에서 자주 등장하는 anti-pattern 의 카탈로그와 각 anti-pattern 의 식별 신호 / 리팩토링 방향을 정의한다.

본 문서가 정의하는 것:
- 12 개의 대표 anti-pattern
- 각 anti-pattern 의 정의 / 식별 신호 / 발생 원인 / 리팩토링 방향
- 코드 냄새와 anti-pattern 의 관계
- 리팩토링 우선순위 결정 기준

본 문서가 정의하지 않는 것:
- 각 리팩토링 기법의 step-by-step — Fowler "Refactoring" 2 판 참고
- 분산 시스템의 anti-pattern — [[../distributed-systems/distributed-systems]]
- CI/CD anti-pattern — [[../software-engineering/software-engineering]] §10

---

## 2. 범위

| 구분 | 포함 |
| --- | --- |
| 대상 | 클래스 / 객체 단위의 설계 anti-pattern |
| 포함 | 리팩토링 시 일반적으로 우선 제거하는 12 가지 |
| 제외 | 일반 코드 냄새 (긴 함수 / 매직 넘버 / 중복) — Fowler 의 카탈로그 |

---

## 3. 용어

| 용어 | 정의 |
| --- | --- |
| **anti-pattern** | 반복적으로 등장하지만 부작용이 큰 설계 / 코드 패턴. 보통 해결책이 함께 정의됨. |
| **code smell** | anti-pattern 의 증거가 되는 코드의 표면 신호 (긴 함수 / 큰 클래스 등). |
| **refactoring** | 외부 동작을 보존하면서 내부 구조를 개선하는 작업. |
| **technical debt** | 의도적 / 비의도적으로 누적된 anti-pattern 의 비용. |

---

## 4. anti-pattern 카탈로그

### 4.1 God Object (God Class)

#### 정의
한 클래스가 너무 많은 책임을 가져 시스템 대부분의 변경이 그 클래스를 거치는 상태.

#### 식별 신호
- 클래스 길이 ≥ 1000 줄
- public method 수 ≥ 30
- field 수 ≥ 15
- 한 PR 이 항상 이 클래스를 수정
- LCOM4 ≥ 5

#### 원인
- 점진적 기능 추가를 한 클래스에 누적
- "Manager" / "Service" 같은 모호한 이름의 시작점

#### 리팩토링
1. method 그룹을 LCOM 으로 식별 (같은 field 쓰는 method 들)
2. 그룹별로 Extract Class
3. 원 클래스는 facade 또는 use case 로 축소

#### 관련
- [[solid-principles]] SRP
- [[coupling-cohesion]] §5

---

### 4.2 Anemic Domain Model

#### 정의
도메인 클래스가 데이터 (field) 만 갖고 행위 (method) 가 없어, 외부 service 가 모든 비즈니스 로직을 가진 상태.

#### 식별 신호
- 도메인 클래스가 getter / setter 만 존재
- Service / Manager 클래스에 비즈니스 로직 집중
- 같은 도메인 invariant 가 여러 service 에 반복 검증
- 도메인 객체의 invalid state 가 자주 발생 (null / negative 등)

#### 원인
- ORM 도구가 만든 entity 를 그대로 사용
- "데이터 = 객체" 라는 단순화
- DTO 와 도메인 객체의 혼동

#### 리팩토링
1. 도메인 클래스에 invariant 검증을 constructor 로 이동
2. setter 제거 → 의도된 state 변경 method 로 대체 (`order.cancel()`)
3. 비즈니스 로직을 service → 도메인 클래스로 이동 (Move Method)
4. 외부 service 는 협력 조율만 남김

#### 관련
- [[good-vs-bad-oop]] §5
- [[concepts]] §7.4
- Eric Evans — DDD "rich domain model"

---

### 4.3 Feature Envy

#### 정의
한 클래스의 method 가 다른 클래스의 데이터에 더 관심이 많은 상태. 거의 모든 작업이 다른 객체의 getter 로 가져오는 형태.

#### 식별 신호
- method 안에서 `other.getX()`, `other.getY()`, `other.getZ()` 가 반복
- 자기 클래스의 field 는 거의 사용 안 함
- 같은 method 가 다른 클래스의 행위처럼 보임

#### 원인
- 책임이 잘못된 클래스에 배치됨
- "데이터는 A, 행위는 B" 식의 분리

#### 리팩토링
- Move Method — method 를 데이터가 있는 클래스로 이동

#### 코드 예
```java
// 나쁨
class Invoice {
    double total(Order o) {
        return o.getItems().stream()
                .mapToDouble(i -> i.getPrice() * i.getQuantity())
                .sum();
    }
}

// 좋음
class Order {
    double total() {
        return items.stream().mapToDouble(Item::subtotal).sum();
    }
}
class Item {
    double subtotal() { return price * quantity; }
}
```

---

### 4.4 Inappropriate Intimacy

#### 정의
두 클래스가 서로의 내부에 너무 깊이 접근하는 상태. encapsulation 경계가 모호.

#### 식별 신호
- 두 클래스가 서로의 private / protected field 에 접근
- 한 클래스의 method 가 다른 클래스의 internal state 를 가정
- 같이 항상 수정되어야 함 (shotgun surgery)

#### 원인
- 책임이 잘못 분배됨
- friend / package-private 의 남용

#### 리팩토링
- Move Method / Move Field
- Extract Class — 공유하는 책임을 새 클래스로
- Change Bidirectional Association to Unidirectional

---

### 4.5 Refused Bequest

#### 정의
자식 클래스가 부모로부터 상속받은 method 의 일부를 사용 안 하거나, `throw UnsupportedOperationException` 으로 거부하는 상태.

#### 식별 신호
- 자식 method 에 `throw new UnsupportedOperationException()`
- 부모의 method 가 자식에서 의미 없음 (Robot.eat())
- 자식이 부모의 일부 행위만 필요

#### 원인
- inheritance 가 잘못 적용 — has-a 인데 is-a 로 모델링
- 인터페이스 분리 부족 (ISP 위반)

#### 리팩토링
- Replace Inheritance with Composition
- Extract Interface — 자식이 필요한 method 만 가진 인터페이스로 분리

#### 관련
- [[composition-over-inheritance]]
- [[solid-principles]] §5 ISP

---

### 4.6 Shotgun Surgery

#### 정의
한 변경이 여러 클래스를 동시에 수정해야 하는 상태. 같은 책임이 여러 곳에 흩어져 있음.

#### 식별 신호
- 1 기능 추가 = 5+ 파일 수정 PR
- 같은 검증 로직이 여러 곳에서 반복
- 같은 enum case 처리가 여러 switch 에서 반복

#### 원인
- 책임이 분산되어 응집이 낮음
- DRY 원칙 위반

#### 리팩토링
- Move Method / Move Field — 책임을 한 클래스로 모음
- Extract Class — 흩어진 책임을 새 클래스로
- Replace Conditional with Polymorphism (switch 폭주 시)

#### 관련 (반대)
- Divergent Change — 한 클래스가 여러 이유로 자주 수정 (SRP 위반)

---

### 4.7 Divergent Change

#### 정의
한 클래스가 서로 다른 이유로 자주 수정되는 상태. Shotgun Surgery 의 반대.

#### 식별 신호
- 같은 클래스가 마케팅 / 재무 / 운영 PR 모두에 등장
- 한 클래스 안 method 들의 변경 이력이 서로 무관

#### 원인
- SRP 위반
- "Service" 같은 모호한 이름

#### 리팩토링
- Extract Class — actor 별로 분리

#### 관련
- [[solid-principles]] §2 SRP

---

### 4.8 Primitive Obsession

#### 정의
도메인 개념을 primitive type (String / int / double) 또는 collection 으로 표현해 의미를 잃는 상태.

#### 식별 신호
- `Money` 를 `double` 로
- `Email` 을 `String` 으로
- 같은 primitive 가 여러 의미로 (`String customerId`, `String orderId`)
- 검증 로직이 사용처마다 반복

#### 원인
- "단순함" 으로 시작 → 도메인 의미 누락 누적
- DTO 와 도메인의 혼동

#### 리팩토링
- Replace Primitive with Value Object — `Email`, `Money`, `CustomerId` 등 value object 도입

#### 코드 예
```java
// 나쁨
void sendEmail(String to, double amount, String currency) { /* */ }

// 좋음
record Email(String value) {
    Email {
        if (!value.contains("@")) throw new IllegalArgumentException();
    }
}
record Money(BigDecimal amount, Currency currency) { /* */ }

void sendEmail(Email to, Money amount) { /* */ }
```

---

### 4.9 Train Wreck (Law of Demeter 위반)

#### 정의
`a.getB().getC().getD().doSomething()` 같이 chain 으로 다른 객체의 깊은 내부를 호출.

#### 식별 신호
- `.` 가 3+ 단계 chain
- chain 중간 객체의 type 변경 시 chain 모두 수정

#### 원인
- 객체가 자기 책임을 외부로 노출
- "묻고 결정하기" (ask, decide) — tell, don't ask 위반

#### 리팩토링
- Hide Delegate — 중간 객체를 외부에 노출하지 않고 method 로 감싸기

#### 코드 예
```java
// 나쁨
boolean canPay = order.getCustomer().getWallet().getBalance().isGreaterThan(price);

// 좋음 (Order 가 안다)
boolean canPay = order.canAfford(price);
```

---

### 4.10 Yo-Yo Problem

#### 정의
상속 계층이 깊어 한 method 의 동작을 추적하려면 여러 부모 / 자식 class 를 오가야 하는 상태.

#### 식별 신호
- 상속 깊이 ≥ 4
- method 의 actual implementation 찾기 어려움
- `super.super.super.foo()` 같은 코드

#### 원인
- inheritance 누적
- "확장성" 명목의 over-engineering

#### 리팩토링
- Flatten Hierarchy
- Replace Inheritance with Composition
- Pull Up Method (공통은 위로) / Push Down Method (특수는 아래로) 로 평탄화

#### 관련
- [[composition-over-inheritance]] §5.4

---

### 4.11 Singleton Abuse (Hidden Global)

#### 정의
Singleton 패턴을 남용해 사실상 전역 상태로 사용. 의존이 숨겨지고 테스트 불가.

#### 식별 신호
- `Foo.getInstance()` 가 비즈니스 로직 곳곳에 등장
- 테스트 시 singleton 의 reset 필요
- 같은 singleton 이 thread 간 race

#### 원인
- "전역 접근의 편리함"
- DI container 미사용

#### 리팩토링
- DI 로 instance 주입
- singleton 은 truly stateless (logger / config) 만 유지

#### 관련
- [[coupling-cohesion]] §4.5 common coupling

---

### 4.12 Big Ball of Mud

#### 정의
모듈 / 계층 / 책임의 경계가 모호한 시스템 전체 수준의 anti-pattern. 위 11 개가 누적된 결과.

#### 식별 신호
- 어디서 어디로 의존하는지 추적 불가
- 신입 onboarding 에 ≥ 2 개월
- "이 부분은 함부로 건들지 마라" 영역 다수

#### 원인
- 단기 deadline 누적
- code review 부재
- 도메인 / 계층 경계 미정의

#### 리팩토링
1. Strangler Fig pattern — 새 코드는 명확한 경계로, 옛 코드는 점진적 교체
2. Bounded Context (DDD) 로 큰 도메인 분리
3. Anti-corruption Layer 로 외부 영향 차단

#### 관련
- Brian Foote / Joseph Yoder — "Big Ball of Mud" (1997)

---

## 5. 코드 냄새 → anti-pattern 매핑

Fowler 의 코드 냄새가 위 anti-pattern 의 표면 신호.

| 코드 냄새 | 가능성 높은 anti-pattern |
| --- | --- |
| Long Method | God Object (작은 단위) |
| Large Class | God Object |
| Long Parameter List | Primitive Obsession, Feature Envy |
| Data Clumps | Primitive Obsession |
| Data Class | Anemic Domain Model |
| Switch Statements | Refused Bequest, polymorphism 부재 |
| Temporary Field | God Object, 책임 혼재 |
| Message Chain | Train Wreck (LoD 위반) |
| Middle Man | 과도한 delegation |
| Inappropriate Intimacy | (동일 anti-pattern) |
| Alternative Classes with Different Interfaces | 인터페이스 부재 — polymorphism 누락 |
| Duplicate Code | DRY 위반 — Shotgun Surgery 의 원인 |

---

## 6. 리팩토링 우선순위

다음 순서로 우선 제거.

| 우선순위 | 이유 |
| --- | --- |
| 1. God Object | 가장 큰 변경 cascade 발생 |
| 2. Anemic Domain Model | invariant 누락의 직접 원인 |
| 3. Train Wreck (LoD 위반) | encapsulation 의 가장 명백한 위반 |
| 4. Feature Envy / Move Method | 책임 재분배의 가장 안전한 시작 |
| 5. Primitive Obsession | 도메인 의미 복구 |
| 6. Refused Bequest | 상속 → 컴포지션 전환 |
| 7. Shotgun Surgery / Divergent Change | 응집 / 결합 재정렬 |
| 8. Yo-Yo Problem | 계층 평탄화 |
| 9. Singleton Abuse | DI 전환 |
| 10. Big Ball of Mud | 가장 큰 작업 — Strangler Fig 로 점진적 |

→ 1-3 은 테스트 작성 후 즉시 시작 가능. 10 은 6 개월 이상의 마이그레이션.

---

## 7. 리팩토링의 전제

| 전제 | 이유 |
| --- | --- |
| 테스트 커버리지 | 동작 보존 검증 가능 |
| 작은 step | 한 번에 1 가지 리팩토링 (Extract / Move / Rename) |
| 컴파일 / 테스트 통과 유지 | 매 step 후 확인 |
| Boy Scout Rule | PR 의 책임 범위 안에서만 |
| 명확한 종료 조건 | "다음 PR 에서 계속" 가능 |

---

## 8. 측정 — 리팩토링의 효과

리팩토링 전후 다음 지표로 효과 검증.

| 지표 | 측정 |
| --- | --- |
| LCOM4 ↓ | 클래스 응집 ↑ |
| fan-out ↓ | 결합 ↓ |
| cyclomatic complexity ↓ | 분기 단순화 |
| cognitive complexity ↓ | 가독성 ↑ |
| PR 평균 파일 수 ↓ | shotgun surgery 감소 |
| test coverage ↑ | 리팩토링 가능한 영역 확대 |
| bug fix lead time ↓ | 영향 범위 축소 |

상세: [[coupling-cohesion]] §6.

---

## 9. 참고

- [[object-oriented-programming|↑ OOP hub]]
- [[concepts]]
- [[solid-principles]]
- [[coupling-cohesion]]
- [[good-vs-bad-oop]]
- [[composition-over-inheritance]]
- [[oop-for-backend]]
- Martin Fowler — "Refactoring" 2판 (2018) — code smell + 리팩토링 카탈로그
- Joshua Kerievsky — "Refactoring to Patterns" (2004)
- Michael Feathers — "Working Effectively with Legacy Code" (2004)
- Brian Foote, Joseph Yoder — "Big Ball of Mud" (1997)
- Sandi Metz — "All the Little Things" (talk, 2014) — refactoring 절차
