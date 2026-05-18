---
title: "coupling and cohesion — 결합도 / 응집도 분류 / 측정"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-18T18:30:00+09:00
tags: [computer-science, oop, coupling, cohesion, metrics]
home_hub: object-oriented-programming
related:
  - "[[object-oriented-programming]]"
  - "[[concepts]]"
  - "[[solid-principles]]"
  - "[[good-vs-bad-oop]]"
  - "[[anti-patterns]]"
---

# coupling and cohesion — 결합도 / 응집도 분류 / 측정

**[[object-oriented-programming|↑ object-oriented-programming]]**

---

## 1. 목적

본 문서는 모듈 간 결합도 (coupling) 와 모듈 내부 응집도 (cohesion) 의 분류 / 측정 / 개선 방법을 정의한다.

본 문서가 정의하는 것:
- 결합도 6 단계 (Stevens / Constantine / Myers 1974) 정의 + 코드 예
- 응집도 7 단계 정의 + 코드 예
- 측정 도구 / 메트릭 (LCOM / fan-in / fan-out / instability)
- 결합 / 응집과 SOLID 의 관계
- 측정값을 어떻게 의사결정에 사용하는가

본 문서가 정의하지 않는 것:
- SOLID 원칙 자체 — [[solid-principles]]
- 안티패턴 카탈로그 — [[anti-patterns]]

---

## 2. 범위

| 구분 | 포함 |
| --- | --- |
| 대상 | OOP 모듈 (class / package / module) |
| 적용 | 일반 SW 공학 — OOP 외에도 적용 가능 |
| 측정 도구 | SonarQube / Checkstyle / Structure101 / NDepend / Code Climate |

---

## 3. 용어

| 용어 | 정의 |
| --- | --- |
| **coupling (결합도)** | 모듈 A 가 모듈 B 에 얼마나 의존하는가의 정도. 낮을수록 좋음. |
| **cohesion (응집도)** | 한 모듈 안 요소들이 얼마나 한 가지 목적에 기여하는가. 높을수록 좋음. |
| **fan-in** | 한 모듈을 의존하는 외부 모듈 수. |
| **fan-out** | 한 모듈이 의존하는 외부 모듈 수. |
| **instability (I)** | `fan-out / (fan-in + fan-out)`. 0=안정 (의존 받음), 1=불안정 (의존 함). |
| **abstractness (A)** | 모듈 안 추상 type (interface / abstract class) 비율. |
| **distance from main sequence (D)** | `|A + I - 1|`. 0 에 가까울수록 좋음 (Martin). |
| **LCOM (Lack of Cohesion of Methods)** | 클래스의 method 가 같은 field 를 공유 안 하는 정도. 높을수록 응집 ↓. |

---

## 4. 결합도 6 단계 (낮음 → 높음)

Stevens / Constantine / Myers (1974) 의 분류. 위에서 아래로 갈수록 나쁨.

### 4.1 data coupling (가장 좋음)

모듈이 필요한 데이터만 인자로 받음.

```java
class TaxCalculator {
    Money calculate(Money amount, double rate) { return amount.multiply(rate); }
}
```

| 평가 | 좋음 |
| --- | --- |
| 변경 영향 | rate 의 source 가 바뀌어도 영향 없음 |
| 테스트 | 인자만 변경 |

### 4.2 stamp coupling (data structure coupling)

모듈이 객체 전체를 받지만 일부만 사용.

```java
class TaxCalculator {
    Money calculate(Order order, double rate) {
        return order.totalPrice().multiply(rate);   // order 의 totalPrice 만 사용
    }
}
```

| 평가 | 보통 |
| --- | --- |
| 변경 영향 | Order class 변경 시 TaxCalculator 가 컴파일에 영향 |
| 개선 | Money 만 받으면 data coupling 으로 감소 |

### 4.3 control coupling

모듈이 다른 모듈의 내부 흐름을 제어하는 flag 전달.

```java
class Reporter {
    void report(Order order, boolean showDetail, boolean exportPdf, boolean sendEmail) {
        // flag 4 개 → 8 가지 경로
    }
}
```

| 평가 | 나쁨 |
| --- | --- |
| 변경 영향 | 새 flag 추가 시 signature 변경 |
| 개선 | flag 대신 polymorphism 또는 별도 method |

### 4.4 external coupling

여러 모듈이 외부 환경 (파일 포맷 / 통신 프로토콜 / 외부 API) 을 공유.

```java
class A { void parse(byte[] data) { /* 외부 프로토콜 v1 가정 */ } }
class B { void send(byte[] data) { /* 외부 프로토콜 v1 가정 */ } }
class C { void log(byte[] data)  { /* 외부 프로토콜 v1 가정 */ } }
// 프로토콜 v2 변경 시 A, B, C 모두 수정
```

| 평가 | 나쁨 |
| --- | --- |
| 개선 | 프로토콜 변환 계층 (anti-corruption layer) 도입 |

### 4.5 common coupling

여러 모듈이 같은 global / mutable state 를 읽고 씀.

```java
class Globals { public static int counter = 0; }

class A { void incr() { Globals.counter++; } }
class B { int get()   { return Globals.counter; } }
```

| 평가 | 매우 나쁨 |
| --- | --- |
| 변경 영향 | A 의 동작이 B 의 결과를 무작위로 변경 |
| 테스트 | global state 초기화 매번 필요 |
| 개선 | 의존 주입 (DI) — singleton 대신 instance 전달 |

### 4.6 content coupling (가장 나쁨)

모듈이 다른 모듈의 내부 구현 / private state 를 직접 접근.

```java
class A {
    public List<Item> items = new ArrayList<>();    // public field
}

class B {
    void modify(A a) {
        a.items.add(0, new Item());        // A 의 내부 직접 변경
        a.items.removeIf(i -> i.expired);
    }
}
```

| 평가 | 최악 |
| --- | --- |
| 변경 영향 | A 의 내부 구조 변경 = B 깨짐 |
| 캡슐화 | 완전 깨짐 |
| 개선 | public field → method 로 캡슐화 |

---

## 5. 응집도 7 단계 (낮음 → 높음)

Stevens / Constantine / Myers + Larman 의 GRASP 결합. 아래로 갈수록 좋음.

### 5.1 coincidental (가장 나쁨)

모듈 안 요소들이 아무 관계 없음. "Utils" / "Helper" 가 흔한 예.

```java
class Utils {
    static String slugify(String s)         { /* */ }
    static long  daysBetween(Date a, Date b){ /* */ }
    static byte[] gzip(byte[] data)         { /* */ }
    static void   sendEmail(String to)      { /* */ }
}
```

→ 4 method 간 공통점 0.

### 5.2 logical

기능적으로 같은 범주에 속하지만 데이터 / 행위 흐름이 다름.

```java
class IOOperation {
    void readFile(String path)  { /* */ }
    void writeDb(Object data)   { /* */ }
    void sendNetwork(String url){ /* */ }
}
```

→ "IO" 라는 범주만 같음. 실제 동작은 분리되어야.

### 5.3 temporal

같은 시점에 실행되어야 하는 동작들의 묶음.

```java
class ApplicationStartup {
    void initLogger()   { /* */ }
    void initDb()       { /* */ }
    void initCache()    { /* */ }
    void startServer()  { /* */ }
}
```

→ "시작 시 실행" 만 공통. 각 책임은 다른 모듈에 분배 가능.

### 5.4 procedural

특정 순서로 실행되어야 함.

```java
class OrderFlow {
    void validate(Order o)   { /* */ }
    void calculate(Order o)  { /* */ }
    void persist(Order o)    { /* */ }
    void notify(Order o)     { /* */ }
}
```

→ 순서 의존이 명확하면 use case 클래스로 정당화 가능.

### 5.5 communicational

같은 데이터를 사용하는 method 들의 묶음.

```java
class OrderReport {
    String renderHtml(Order o) { /* */ }
    String renderPdf(Order o)  { /* */ }
    String renderCsv(Order o)  { /* */ }
}
```

→ 같은 Order 를 다른 포맷으로 표현. 책임 분리 후보지만 communicational 만으로도 정당화 가능.

### 5.6 sequential

한 method 의 출력이 다음 method 의 입력.

```java
class ImagePipeline {
    BufferedImage load(File f)              { /* */ }
    BufferedImage resize(BufferedImage img) { /* */ }
    BufferedImage compress(BufferedImage img){ /* */ }
    void save(BufferedImage img, File f)    { /* */ }
}
```

→ pipeline pattern. 적절.

### 5.7 functional (가장 좋음)

모듈의 모든 요소가 하나의 명확한 목적을 위해 협력.

```java
class Money {
    private final BigDecimal amount;
    private final Currency currency;

    Money(BigDecimal amount, Currency currency) { /* */ }
    Money plus(Money other)              { /* */ }
    Money minus(Money other)             { /* */ }
    Money multiply(double rate)          { /* */ }
    boolean isGreaterThan(Money other)   { /* */ }
}
```

→ 모든 method 가 "금액 표현 / 산술" 이라는 단일 목적에 기여.

---

## 6. 측정 메트릭

### 6.1 LCOM (Lack of Cohesion of Methods)

LCOM4 가 일반적 (Hitz / Montazeri).

```
LCOM4 = 같은 field 를 공유 안 하는 method 쌍의 그래프 컴포넌트 수
```

```java
class Mixed {
    private int a, b;
    void useA1() { a++; }
    void useA2() { a--; }
    void useB1() { b++; }
    void useB2() { b--; }
}
// LCOM4 = 2 (a 만 쓰는 method 2 + b 만 쓰는 method 2 → 분리 가능)
```

| LCOM4 | 의미 |
| --- | --- |
| 1 | 응집 OK (모든 method 가 연결) |
| 2+ | 분리 후보 — 각 컴포넌트가 별도 class 가 될 수 있음 |

### 6.2 fan-in / fan-out

| 메트릭 | 의미 | 권장 |
| --- | --- | --- |
| fan-in | 이 모듈을 의존하는 모듈 수 | 핵심 도메인 = 높음 |
| fan-out | 이 모듈이 의존하는 모듈 수 | 낮을수록 좋음 (보통 ≤ 7) |

→ high fan-in + low fan-out = stable. 핵심 도메인 모델이 이 위치.

→ high fan-out + low fan-in = unstable. 외부 어댑터 / use case 가 이 위치.

### 6.3 instability (Robert Martin)

```
I = fan-out / (fan-in + fan-out)
```

| I | 의미 |
| --- | --- |
| 0 | 완전 안정 — 다른 모듈이 의존, 자신은 의존 안 함 |
| 1 | 완전 불안정 — 자신이 의존, 다른 모듈이 자신을 의존 안 함 |

### 6.4 abstractness

```
A = abstract type 수 / 전체 type 수
```

| A | 의미 |
| --- | --- |
| 0 | 모두 concrete |
| 1 | 모두 abstract (interface only) |

### 6.5 distance from main sequence

```
D = |A + I - 1|
```

| D | 의미 |
| --- | --- |
| 0 | 이상적 (안정 모듈은 추상, 불안정 모듈은 구체) |
| 0.7+ | 안티패턴 — "안정 + 구체" (변경 위험 큰 의존 핵심) 또는 "불안정 + 추상" (쓸데없는 추상화) |

---

## 7. 결합도 ↔ 응집도 트레이드오프

두 지표는 독립이 아니라 같은 동전의 양면.

| 상태 | 의미 |
| --- | --- |
| **고결합 + 저응집** | 큰 god 클래스가 모든 것을 알고 모든 것에 의존 — 최악 |
| **고결합 + 고응집** | 한 책임에 집중하지만 너무 많은 외부 의존 — 격리 어려움 |
| **저결합 + 저응집** | 작게 잘 쪼개졌으나 한 클래스 안 method 가 무관 — 분할 자체가 잘못 |
| **저결합 + 고응집** | 이상 — 각 클래스가 한 책임 + 외부 의존 최소 |

→ 목표: 저결합 + 고응집. SOLID 의 SRP / DIP 가 직접 기여.

---

## 8. SOLID 와의 매핑

| SOLID | 결합 / 응집 기여 |
| --- | --- |
| SRP | 응집도 ↑ (한 책임 = 한 모듈) |
| OCP | 결합도 ↓ (확장이 추상에 의존 → 기존 코드 영향 없음) |
| LSP | 결합도 ↓ (자식이 부모 대체 가능 → 호출자가 자식 type 몰라도 됨) |
| ISP | 결합도 ↓ (작은 인터페이스 = 클라이언트가 필요 method 만 의존) |
| DIP | 결합도 ↓ (추상에 의존 → 구체 변경 영향 0) |

→ SOLID = 결합도 ↓ + 응집도 ↑ 의 구체 실행 지침.

---

## 9. 측정값의 사용

### 9.1 의사결정 매트릭스

| 측정 값 | 의사결정 |
| --- | --- |
| 한 class 의 LCOM4 ≥ 3 | 분리 후보 — SRP 위반 의심 |
| 한 module 의 fan-out ≥ 10 | 분할 후보 — 너무 많은 외부 의존 |
| 한 module 의 D ≥ 0.7 | 위치 재검토 (핵심 도메인이 너무 구체적이거나, 어댑터가 너무 추상적) |
| circular dependency 발견 | 즉시 해결 (interface 추출, dependency inversion) |
| package fan-in 으로 0 (사용 안 됨) | dead code — 제거 검토 |

### 9.2 도구

| 도구 | 측정 |
| --- | --- |
| SonarQube | LCOM / cyclomatic / cognitive / coverage |
| Structure101 / NDepend | package 의존 그래프 / instability / abstractness |
| jdepend (Java) | A / I / D |
| pylint / radon (Python) | LCOM / cyclomatic |
| dependency-cruiser (JS/TS) | module 의존 그래프 + circular detection |

### 9.3 측정의 한계

| 한계 | 보완 |
| --- | --- |
| 메트릭이 좋아도 코드가 나쁠 수 있음 | 코드 리뷰와 결합 |
| 메트릭이 나빠도 정당화 가능 (legacy / migration 중) | 의도된 예외 인정 |
| 메트릭 최적화 자체가 목표가 되면 over-engineering | 도메인 / 변경 빈도 기준 우선 |

---

## 10. 흔한 실패 모드

| 실패 | 원인 | 정정 |
| --- | --- | --- |
| Utils / Helper / Common 클래스가 점점 비대 | coincidental cohesion | 도메인 별 클래스로 분배 |
| god class (3000 줄 + 50 method) | 저응집 + 고결합 누적 | LCOM 측정 → 컴포넌트별 분리 |
| 한 PR 이 10+ 파일 수정 | 결합 cascade | 책임 재분배 + interface |
| circular dependency | 모듈 간 양방향 의존 | 한쪽에 interface 추출, 다른 쪽이 구현 |
| 같은 데이터를 여러 클래스가 다르게 검증 | 결합 + 응집 모두 깨짐 | 데이터 + 검증을 한 클래스 (encapsulation) |
| 전역 singleton 의존 | common coupling | DI 로 instance 주입 |
| public field 직접 변경 | content coupling | private + method |

---

## 11. 참고

- [[object-oriented-programming|↑ OOP hub]]
- [[concepts]]
- [[solid-principles]]
- [[good-vs-bad-oop]]
- [[anti-patterns]]
- [[../software-engineering/software-engineering]] §11 (코드 메트릭)
- Stevens, Myers, Constantine — "Structured Design" (IBM Systems Journal, 1974) — coupling / cohesion 원전
- Robert C. Martin — "Clean Architecture" (2017) §14 — package metrics (I / A / D)
- Hitz, Montazeri — "Measuring Coupling and Cohesion in Object-Oriented Systems" (1995)
- Sonar Way — https://www.sonarsource.com (default rule set)
