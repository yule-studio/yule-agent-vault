---
title: "디자인 패턴 (Design Patterns) — Hub"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T12:30:00+09:00
tags: [design-patterns, gof, hub, creational, structural, behavioral]
---

# 디자인 패턴 (Design Patterns) — Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 단일 사전 |
| v.2.0.0 | 2026-05-13 | engineering-agent/tech-lead | 각 패턴 별 폴더 + 언어/프레임워크별 적용 노트로 재편 |

**[[../computer-science|↑ computer-science]]**

> 반복되는 설계 문제에 대한 검증된 해결책의 카탈로그. 각 GoF 23 패턴은 별도 폴더에 hub + 언어별 (Java/Spring · Python/Django/FastAPI · TypeScript/Node/React · Go) 적용 노트로 분해.

---

## 0. 한 줄

**"이 문제는 이 패턴" 의 검증된 매핑.** 1994 GoF 가 23 개 OOP 패턴 카탈로그화. 이후 PoEAA (엔터프라이즈) / 분산 / DDD / FP 로 확장.

---

## 1. 패턴 카테고리

### 1.1 생성 (Creational, 5) — 객체 생성 방법

| 패턴 | 의도 | 폴더 |
| --- | --- | --- |
| **Singleton** | 한 인스턴스 | [[singleton/singleton]] |
| **Factory Method** | 자식이 만들 타입 결정 | [[factory-method/factory-method]] |
| **Abstract Factory** | 관련 객체 가족 | [[abstract-factory/abstract-factory]] |
| **Builder** | 복잡한 객체 단계별 | [[builder/builder]] |
| **Prototype** | 복제 | [[prototype/prototype]] |

### 1.2 구조 (Structural, 7) — 객체 합성

| 패턴 | 의도 | 폴더 |
| --- | --- | --- |
| **Adapter** | 인터페이스 호환 | [[adapter/adapter]] |
| **Bridge** | 추상과 구현 분리 | [[bridge/bridge]] |
| **Composite** | 부분-전체 트리 | [[composite/composite]] |
| **Decorator** | 동적 책임 추가 | [[decorator/decorator]] |
| **Facade** | 단순 인터페이스 | [[facade/facade]] |
| **Flyweight** | 공유로 메모리 절약 | [[flyweight/flyweight]] |
| **Proxy** | 대리자 (lazy / remote / protection) | [[proxy/proxy]] |

### 1.3 행위 (Behavioral, 11) — 상호작용

| 패턴 | 의도 | 폴더 |
| --- | --- | --- |
| **Chain of Responsibility** | 처리자 체인 | [[chain-of-responsibility/chain-of-responsibility]] |
| **Command** | 요청을 객체로 | [[command/command]] |
| **Iterator** | 순회 추상화 | [[iterator/iterator]] |
| **Mediator** | 상호작용 중개 | [[mediator/mediator]] |
| **Memento** | 상태 저장 / 복원 | [[memento/memento]] |
| **Observer** | 1-N 통지 | [[observer/observer]] |
| **State** | 상태 별 행동 | [[state/state]] |
| **Strategy** | 알고리즘 교체 | [[strategy/strategy]] |
| **Template Method** | 알고리즘 골격 | [[template-method/template-method]] |
| **Visitor** | 구조 + 연산 분리 | [[visitor/visitor]] |
| **Interpreter** | 문법 평가 | [[interpreter/interpreter]] |

---

## 2. 각 패턴 폴더 구조

```
<pattern>/
├── <pattern>.md         ← hub: 의도, 언제 쓰는가, 구조, 비교, 함정
├── <pattern>-java.md    ← Java 기본 + Spring / Jakarta 적용
├── <pattern>-python.md  ← Python + Django / FastAPI / std 적용
├── <pattern>-typescript.md ← TypeScript + Node / React / NestJS 적용
└── <pattern>-go.md      ← Go idiomatic + std library 적용
```

---

## 3. 패턴 선택 트리 — 짧은 가이드

### 객체 생성이 복잡한가?
- 한 인스턴스만 필요 → **Singleton** (다만 DI 가 보통 더 낫다)
- 자식이 어떤 타입 만들지 결정 → **Factory Method**
- 관련 객체 그룹 (Win/Mac UI 등) → **Abstract Factory**
- 매개변수 많은 객체 단계별 → **Builder**
- 비싼 객체를 자주 만든다 → **Prototype**

### 인터페이스가 안 맞는가?
- 두 인터페이스 호환 → **Adapter**
- 클래스 폭발 (속성 × 행동) → **Bridge**
- 부분-전체 (트리) → **Composite**

### 행동을 동적으로 바꾸고 싶은가?
- 알고리즘 교체 → **Strategy**
- 상태 별 행동 → **State**
- 책임 추가 / 제거 → **Decorator**
- 처리 순서 chain → **Chain of Responsibility**

### 통신이 복잡한가?
- 1-N 변경 통지 → **Observer**
- 다대다 → **Mediator**
- 요청을 객체화 (undo / queue) → **Command**

### 표준 알고리즘인가?
- 알고리즘 골격 유지 + 일부 step 만 자식 → **Template Method**
- 객체에 새 연산 추가 (구조 안 바꿈) → **Visitor**

---

## 4. 패턴 비교 (자주 헷갈리는)

| 비교 | 차이 |
| --- | --- |
| **Adapter vs Facade** | Adapter = 인터페이스 변환. Facade = 단순화. |
| **Factory Method vs Abstract Factory** | FM = 한 객체 생성. AF = 객체 가족. |
| **Strategy vs State** | Strategy = 외부가 교체. State = 내부가 자동 전환. |
| **Decorator vs Proxy** | Decorator = 추가. Proxy = 대리 (제어). |
| **Template Method vs Strategy** | TM = 상속. Strategy = 합성. |
| **Bridge vs Adapter** | Bridge = 설계 시 분리. Adapter = 사후 호환. |

---

## 5. 함수형 / 현대 언어에서의 GoF

| GoF | 함수형 대응 |
| --- | --- |
| Strategy | High-order function |
| Command | Closure |
| Iterator | Lazy sequence / generator |
| Observer | Stream / Reactive (Rx) |
| Visitor | Pattern matching |
| Template Method | Higher-order function |
| Decorator | Function composition |
| State | Monad / Reducer |
| Singleton | Module (Python module / Go package) |
| Factory | Constructor function |

→ 동적 / 함수형 언어에서는 일부 패턴이 "그냥 언어 기능".

---

## 6. 엔터프라이즈 / 분산 / 동시성 패턴 (별도 노트)

- **PoEAA (Fowler 2002)**: Transaction Script / Domain Model / Active Record / Data Mapper / Repository / Unit of Work / DTO / Front Controller / MVC.
- **동시성**: Producer-Consumer / Reader-Writer / Thread Pool / Future / Actor / CSP / Reactor / Proactor.
- **분산 / MSA**: Service Discovery / Circuit Breaker / Bulkhead / Saga / CQRS / Event Sourcing / Outbox / Sidecar / API Gateway / BFF / Strangler Fig.

자세히는 [[../distributed-systems/distributed-systems#10 분산 패턴]] 참조.

---

## 7. 안티패턴 (총괄)

| | |
| --- | --- |
| **패턴 사냥** | "패턴 적용" 이 목표가 됨. 문제부터. |
| **책에 있는 그대로** | 언어 / 컨텍스트에 맞게 변형 필요. |
| **이름만 알고 의도 모름** | "이건 Strategy 패턴" 만 알고 왜 인지 모름. |
| **패턴 = 좋은 코드 X** | 나쁜 코드에 패턴 적용해도 여전히 나쁨. |
| **너무 많은 패턴 동시 적용** | 신규 코드 가독성 ↓. |
| **Singleton 남용** | 글로벌 상태 = 테스트 지옥. DI 로. |
| **모든 곳에 Factory** | 단순 생성에 팩토리 X — YAGNI. |
| **깊은 Decorator 체인** | 디버깅 / 추적 어려움. |
| **Visitor 의 강제** | 함수형 매칭이 있는 언어에선 불필요. |

---

## 8. 면접 질문 (요약)

1. **Singleton 의 문제와 대안** — `[[singleton/singleton]]`
2. **Factory vs Abstract Factory vs Builder** — `[[factory-method/factory-method]]` / `[[abstract-factory/abstract-factory]]` / `[[builder/builder]]`
3. **Decorator vs Proxy** — `[[decorator/decorator]]` / `[[proxy/proxy]]`
4. **Observer 의 단점** — `[[observer/observer]]`
5. **Strategy vs State** — `[[strategy/strategy]]` / `[[state/state]]`
6. **Template Method vs Strategy** — `[[template-method/template-method]]` / `[[strategy/strategy]]`
7. **Adapter vs Facade** — `[[adapter/adapter]]` / `[[facade/facade]]`

---

## 9. 학습 자료

- **Design Patterns** (GoF, 1994) — 원전.
- **Head First Design Patterns** — 시각적.
- **Refactoring to Patterns** (Joshua Kerievsky).
- **Patterns of Enterprise Application Architecture** (Fowler 2002).
- **Game Programming Patterns** (Robert Nystrom) — 무료 https://gameprogrammingpatterns.com.
- **refactoring.guru** — https://refactoring.guru/design-patterns — 시각화 우수.
- **Patterns of Distributed Systems** (Joshi).

---

## 10. 관련

- [[../software-engineering/software-engineering]] — SOLID
- [[../distributed-systems/distributed-systems]] — MSA / 분산 패턴
- [[../computer-science|↑ computer-science]]
