---
title: "객체지향 프로그래밍 (Object-Oriented Programming) — Hub"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-18T16:00:00+09:00
tags: [computer-science, oop, hub]
home_hub: computer-science
related:
  - "[[../computer-science]]"
  - "[[concepts]]"
  - "[[solid-principles]]"
  - "[[good-vs-bad-oop]]"
  - "[[composition-over-inheritance]]"
  - "[[coupling-cohesion]]"
  - "[[anti-patterns]]"
  - "[[oop-for-backend]]"
  - "[[tell-dont-ask-and-law-of-demeter]]"
  - "[[value-objects]]"
  - "[[ddd-tactical-patterns]]"
  - "[[../design-patterns/design-patterns]]"
  - "[[../software-engineering/software-engineering]]"
  - "[[../../../40-patterns/architecture-patterns/architecture-patterns]]"
  - "[[../../../40-patterns/refactoring-patterns/refactoring-patterns]]"
---

# 객체지향 프로그래밍 (Object-Oriented Programming) — Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-18 | engineering-agent/tech-lead | 최초 — OOP 영역 진입점 + 8 노트 인덱스 |
| v.1.1.0 | 2026-05-19 | engineering-agent/tech-lead | 3 깊이 노트 추가 (tell-don't-ask / value-objects / ddd-tactical-patterns) + 40-patterns/architecture-patterns 영역 link |

**[[../computer-science|↑ computer-science]]**

---

## 1. 영역 정의

객체지향 프로그래밍 (OOP) 은 **상태와 행위를 가진 객체** 를 기본 단위로 시스템을 구성하는 프로그래밍 패러다임이다. 객체끼리는 메시지로 통신하며 변경의 영향 범위는 객체의 경계 안으로 제한된다.

본 영역은 OOP 의 정의 / 원칙 / 평가 기준 / 적용 방법을 다룬다. 구체 GoF 패턴은 [[../design-patterns/design-patterns]], 일반 SW 공학 원칙은 [[../software-engineering/software-engineering]] 에서 다룬다.

---

## 2. 본 영역이 정의하는 것 / 정의하지 않는 것

| 정의함 | 정의 안 함 |
| --- | --- |
| 객체 / 클래스 / 메시지 / 인터페이스의 의미 | 특정 언어 syntax — 각 언어별 별도 노트 |
| 4 pillar (encapsulation / inheritance / polymorphism / abstraction) | UML / 다이어그램 작성법 |
| SOLID / coupling / cohesion / tell-don't-ask | GoF 23 패턴 카탈로그 — [[../design-patterns/design-patterns]] |
| 좋은 OOP vs 나쁜 OOP 의 객관적 식별 기준 | TDD / 테스트 전략 — [[../software-engineering/software-engineering]] |
| OOP anti-pattern + 리팩토링 방향 | 도메인 모델링 일반론 — DDD ([[../software-engineering/software-engineering#5.4]]) |

---

## 3. 노트 인덱스

| 노트 | 다루는 주제 |
| --- | --- |
| [[concepts]] | 4 pillar (encapsulation / inheritance / polymorphism / abstraction) + class / object / message / interface |
| [[solid-principles]] | SRP / OCP / LSP / ISP / DIP 5 원칙의 정의 / 위반 / 적용 |
| [[good-vs-bad-oop]] | 좋은 OOP / 나쁜 OOP 의 식별 표 + before/after 코드 비교 |
| [[composition-over-inheritance]] | 상속의 한계와 컴포지션의 적용 기준 |
| [[coupling-cohesion]] | 결합도 6 단계 / 응집도 7 단계 + 측정 방법 |
| [[anti-patterns]] | OOP anti-pattern 카탈로그 (God Object / Anemic / Feature Envy / 기타) + 리팩토링 방향 |
| [[oop-for-backend]] | 백엔드 프레임워크 (Spring / Django / Rails) 에서 OOP 적용 |
| [[tell-dont-ask-and-law-of-demeter]] | 두 통신 원칙 — 묻지 말고 시켜라 + 친구하고만 대화 + Hide Delegate |
| [[value-objects]] | identity 없는 immutable 도메인 객체 + primitive obsession 해소 |
| [[ddd-tactical-patterns]] | Entity / VO / Aggregate / Repository / Domain Service / Domain Event / Factory |

> 인접 영역 (재사용 설계 패턴): [[../../../40-patterns/architecture-patterns/architecture-patterns|↗ 40-patterns/architecture-patterns]] (Layered / Hexagonal / Clean), [[../../../40-patterns/refactoring-patterns/refactoring-patterns|↗ 40-patterns/refactoring-patterns]] (Replace Conditional with Polymorphism)

---

## 4. 학습 순서

1. [[concepts]] — 용어와 4 pillar 의 정의를 확정한다.
2. [[coupling-cohesion]] — OOP 의 모든 원칙이 결합/응집의 트레이드오프임을 이해한다.
3. [[solid-principles]] — 5 원칙을 개별로 본다.
4. [[tell-dont-ask-and-law-of-demeter]] — 객체 간 통신의 두 원칙으로 캡슐화를 강화한다.
5. [[composition-over-inheritance]] — inheritance 의 함정과 composition 의 대안을 본다.
6. [[value-objects]] — primitive obsession 해소 + immutable 도메인 표현.
7. [[anti-patterns]] — 실제 코드에서 자주 나타나는 안티패턴을 식별한다.
8. [[good-vs-bad-oop]] — 좋은 / 나쁜 OOP 를 동일 도메인의 코드로 비교한다.
9. [[ddd-tactical-patterns]] — Aggregate 중심의 도메인 모델링으로 확장한다.
10. [[oop-for-backend]] — 프레임워크 안에서 위 원칙을 적용한다.
11. [[../../../40-patterns/architecture-patterns/architecture-patterns]] — Layered / Hexagonal / Clean 으로 application 구조화.
12. [[../design-patterns/design-patterns]] — 구체적 패턴 카탈로그로 확장한다.

---

## 5. 좋은 OOP 의 정의 — 본 영역의 평가 기준

본 영역의 모든 노트는 "좋은 OOP" 를 다음 5 가지 기준으로 평가한다.

| 기준 | 정의 |
| --- | --- |
| **명확한 책임 (cohesion)** | 한 객체가 단일하고 식별 가능한 책임을 가진다. |
| **낮은 결합 (coupling)** | 객체 간 변경이 다른 객체로 전파되는 범위가 작다. |
| **선언적 행위 (tell, don't ask)** | 클라이언트는 객체에게 일을 시키지, 상태를 묻고 결정하지 않는다. |
| **대체 가능성 (substitutability)** | 같은 인터페이스의 다른 구현으로 교체 시 동작이 보존된다 (Liskov). |
| **변경 비용 한정 (locality of change)** | 요구 변경 1 건이 수정하는 클래스 수가 적다. |

세부 진술: [[good-vs-bad-oop]] §3.

---

## 6. OOP 영역의 외부 참조

| 외부 | 본 영역과의 관계 |
| --- | --- |
| [[../design-patterns/design-patterns]] | OOP 의 원칙을 구체 패턴으로 카탈로그화 (GoF 23 + 확장) |
| [[../software-engineering/software-engineering]] | OOP 외 SW 공학 원칙 (KISS / DRY / YAGNI / 테스트 / 리팩토링 / CI/CD) |
| [[../distributed-systems/distributed-systems]] | OOP 의 객체 모델이 분산 환경 (Actor / message-passing) 에서 확장되는 방식 |
| [[../../backend|↗ backend]] | 본 영역의 원칙이 적용되는 백엔드 코드베이스 |

---

## 7. 외부 표준 / 참고 문헌

| 자료 | 영역 |
| --- | --- |
| Smalltalk-80 (Goldberg / Robson 1983) | OOP 의 원형 — message passing 정의 |
| Design Patterns (Gamma / Helm / Johnson / Vlissides 1994) | GoF 23 패턴 |
| Refactoring (Fowler, 2nd ed 2018) | 코드 냄새 + 리팩토링 카탈로그 |
| Clean Code (Martin 2008) | 객체 / 함수 / 이름 + SOLID |
| Object-Oriented Software Construction (Meyer 1997) | Design by Contract |
| A Philosophy of Software Design (Ousterhout 2018) | 깊이의 모듈 / 정보 은닉 |
| Working Effectively with Legacy Code (Feathers 2004) | 리팩토링 → 테스트 가능 OOP |
| Implementing Domain-Driven Design (Vernon 2013) | OOP 위의 도메인 모델 |

---

## 8. 본 영역의 유지보수

| 트리거 | 갱신 항목 |
| --- | --- |
| 새 anti-pattern / 원칙 추가 | §3 인덱스 + 해당 노트 |
| 평가 기준 변경 | §5 + [[good-vs-bad-oop]] §3 |
| 학습 순서 변경 | §4 |
| 외부 영역 cross-link 추가 | §6 |

---

## 9. 관련

- [[../computer-science|↑ computer-science]]
- [[concepts]] · [[solid-principles]] · [[good-vs-bad-oop]] · [[composition-over-inheritance]] · [[coupling-cohesion]] · [[anti-patterns]] · [[oop-for-backend]]
- [[../design-patterns/design-patterns]]
- [[../software-engineering/software-engineering]]
- [[../distributed-systems/distributed-systems]]
