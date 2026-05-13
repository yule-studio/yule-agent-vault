---
title: "Factory Method — Hub"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, gof, creational, factory-method]
---

# Factory Method — Hub

**[[../design-patterns|↑ 디자인 패턴]]** · GoF Creational

## 1. 한 줄

**객체 생성을 자식 클래스가 결정하게 위임.** 어떤 인스턴스를 만들지 호출자가 모르게.

## 2. 의도

- 상위 클래스가 "어떤 인터페이스의 객체가 필요" 라고만 표현.
- 자식이 "어떤 구체 타입" 인지 결정.
- new 의 결합을 인터페이스로 우회.

## 3. 구조

```
   Creator(abstract)         Product(interface)
   ─────────────             ─────────────────
   +factoryMethod() ─────►   + operation()
   +operation()              ▲
   ▲                         │
   │                  ConcreteProductA  ConcreteProductB
ConcreteCreatorA
ConcreteCreatorB
```

## 4. 언제 쓰는가

- **객체 종류가 런타임 / 설정에 따라 결정** — driver / parser / serializer 선택.
- **테스트** — production / fake 객체 swap.
- **plug-in 아키텍처** — 외부 jar / package 가 자기 객체 등록.
- **subclass 가 특정 구현 결정** — framework 의 hook.

## 5. 언제 안 쓰는가 (안티)

- **단순 객체 1 종** — `new Foo()` 가 더 명확.
- **factory 자체가 거대 if-else** — 추가 abstraction 없이 분기만.
- **YAGNI** — 미래의 유연성을 위해 미리 만들지 마.

## 6. Factory Method vs Abstract Factory vs Builder

| | Factory Method | Abstract Factory | Builder |
| --- | --- | --- | --- |
| 만드는 것 | 한 객체 | 객체 가족 | 복잡 객체 단계 |
| 결정자 | subclass | factory class | 호출 chain |
| 예 | DocumentParser | UIFactory (button + checkbox) | HttpRequest |

## 7. 언어 별

- [[factory-method-java|Java / Spring]]
- [[factory-method-python|Python / Django / FastAPI]]
- [[factory-method-typescript|TypeScript / Node / NestJS]]
- [[factory-method-go|Go]]

## 8. 실 프레임워크 예

- **JDBC `DriverManager.getConnection`** — URL 로 driver 선택.
- **Spring `BeanFactory` / `ApplicationContext`** — bean factory method.
- **Python `__init_subclass__` + plugin registry**.
- **Go `database/sql.Open(driverName, ...)`**.
- **React `createElement`**.

## 9. 함정

1. **factory 가 거대 switch / if 가 됨** — 객체 종류 추가마다 factory 수정. → registry 패턴.
2. **circular dep** — factory ↔ product.
3. **테스트 시 mock factory 못 끼움** — DI 로 factory 인터페이스 분리.
4. **factory 안 lazy init 의 thread-safety** — `sync.Once` / `synchronized`.

## 10. 관련

- [[../abstract-factory/abstract-factory]] — 객체 가족.
- [[../builder/builder]] — 복잡 객체 단계 빌딩.
- [[../singleton/singleton]] — factory 가 자체 singleton 흔함.
