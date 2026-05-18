---
title: "architecture-patterns — Hub"
kind: knowledge
project: patterns
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-18T21:30:00+09:00
tags: [patterns, architecture, hub]
home_hub: patterns
related:
  - "[[../patterns]]"
  - "[[pattern-hexagonal-architecture]]"
  - "[[pattern-clean-architecture]]"
  - "[[pattern-layered-architecture]]"
  - "[[../../20-areas/computer-science/object-oriented-programming/oop-for-backend]]"
---

# architecture-patterns — Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-18 | engineering-agent/tech-lead | 최초 — 카테고리 hub + 3 패턴 |

**[[../patterns|↑ 40-patterns]]**

---

## 1. 영역 정의

본 영역은 백엔드 / 응용 서비스의 **모듈 / 계층 / 의존 방향** 을 정의하는 재사용 가능한 architecture pattern 의 카탈로그다.

본 영역이 정의하는 것:
- application 내부 구조 (계층 / 모듈 / 의존 방향) 의 표준 패턴
- 각 패턴의 적용 조건 / 비용 / 트레이드오프
- 같은 도메인을 여러 패턴으로 모델링한 비교 예제

본 영역이 정의하지 않는 것:
- 분산 / 시스템 아키텍처 (MSA / event-driven / CQRS / Saga) — [[../../20-areas/computer-science/distributed-systems/distributed-systems]]
- 배포 / 인프라 패턴 — [[../deployment-patterns/]]
- API 통신 패턴 (REST / GraphQL / gRPC) — [[../api-patterns/]]

---

## 2. 패턴 인덱스

| 패턴 | 노트 | 한 줄 정의 |
| --- | --- | --- |
| Layered (N-Tier) | [[pattern-layered-architecture]] | Controller / Service / Repository / Domain 의 표준 4 계층 |
| Hexagonal (Ports & Adapters) | [[pattern-hexagonal-architecture]] | 도메인 중심 + 외부 (DB / UI / API) 는 adapter 로 격리 |
| Clean Architecture | [[pattern-clean-architecture]] | Uncle Bob 의 4 동심원 + dependency rule (안쪽으로만) |

---

## 3. 패턴 결정 매트릭스

| 조건 | 권장 패턴 |
| --- | --- |
| 단순 CRUD, 빠른 출시 | Layered |
| 도메인이 복잡, 외부 의존 다수 (DB / 메시지 큐 / 외부 API) | Hexagonal |
| 도메인이 매우 복잡 + 장기 유지보수 + 인프라 교체 가능성 | Clean Architecture |
| 마이크로서비스 단위 분리 필요 | Hexagonal + bounded context (각 서비스) |
| 팀 규모 작음 + 도메인 단순 | Layered (over-engineering 회피) |
| 팀 규모 큼 + 다중 도메인 + 다양한 클라이언트 (web / mobile / api) | Hexagonal 또는 Clean |
| event sourcing / CQRS 도입 | Hexagonal + read model 분리 |

→ 세 패턴은 점진적으로 적용 가능. Layered → Hexagonal → Clean.

---

## 4. 공통 원칙

세 패턴 모두 다음 원칙을 다른 방식으로 구현한다.

| 원칙 | 의미 |
| --- | --- |
| **도메인 중심** | 비즈니스 규칙이 외부 (UI / DB / framework) 의 변경에 영향받지 않음 |
| **의존 방향** | 의존이 항상 도메인 (안쪽) 으로 향함 |
| **포트 / 추상화** | 외부와의 접점은 interface 로 표현 |
| **테스트 가능성** | 도메인 로직이 외부 의존 없이 테스트 가능 |

---

## 5. 패턴 비교 — 같은 도메인의 모델링 위치

같은 "주문 생성" 기능이 각 패턴에서 어디에 위치하는가.

| 책임 | Layered | Hexagonal | Clean |
| --- | --- | --- | --- |
| HTTP 진입 | Controller | Driving Adapter (HTTP) | Interface Adapter (Controller) |
| 흐름 조율 | Service | Application Service | Use Case |
| 비즈니스 규칙 | Service or Entity | Domain (Aggregate) | Entity + Use Case Interactor |
| 영속화 추상 | Repository interface | Port (Driven) | Interface Adapter (Boundary) |
| 영속화 구현 | Repository impl | Driven Adapter (DB) | Frameworks / Drivers |

→ 이름과 layer 깊이만 다르고, 본질은 같음: **도메인을 외부로부터 격리** + **의존 방향이 도메인으로**.

---

## 6. 적용 비용

각 패턴의 적용에 드는 비용 / 코드량 / 학습 곡선.

| 항목 | Layered | Hexagonal | Clean |
| --- | --- | --- | --- |
| 학습 곡선 | 낮음 | 중 | 높음 |
| 초기 코드량 | 적음 | 보통 | 많음 (4 계층 + interface 다수) |
| 변경 비용 | 낮음 (단순) | 중 | 낮음 (격리 잘됨) |
| 인프라 교체 가능성 | 낮음 | 높음 | 높음 |
| 단위 테스트 작성 비용 | 보통 | 낮음 | 낮음 |
| 신규 개발자 onboarding | 빠름 | 보통 | 느림 |
| 작은 변경의 PR 크기 | 작음 | 보통 | 큼 |

---

## 7. 흔한 실패 모드

| 실패 | 원인 | 정정 |
| --- | --- | --- |
| Hexagonal / Clean 을 모든 프로젝트에 강제 | over-engineering | 단순 CRUD 는 Layered |
| Layered 에서 Service 가 god class 화 | SRP 위반 + 적절한 도메인 모델 부재 | Use Case 단위 분리 + 도메인에 행위 이동 |
| Hexagonal 의 Port 가 너무 많음 | 모든 메서드를 interface 로 | 실제 교체 가능성 있는 곳만 |
| Clean Architecture 의 Entity 가 anemic | 도메인 로직이 Use Case 에 흡수 | Entity 안에 invariant + 행위 |
| 도메인 객체에 framework annotation | 도메인 ↔ 인프라 결합 | 별도 ORM Entity + Mapper |
| 패턴 이름은 따르지만 실제 의존 방향 위반 | 형식만 적용 | dependency-cruiser / ArchUnit 으로 검증 |

---

## 8. 검증 도구

| 도구 | 검증 |
| --- | --- |
| ArchUnit (Java) | 의존 방향 / 패키지 규칙 |
| dependency-cruiser (JS/TS) | 모듈 그래프 / circular detection |
| Structure101 / NDepend | 계층 의존 시각화 |
| pre-commit hook + grep | 도메인이 외부 import 안 함 검증 |

---

## 9. 외부 표준 / 참고

| 자료 | 영역 |
| --- | --- |
| "Patterns of Enterprise Application Architecture" (Fowler 2002) | Service Layer / Domain Model / Transaction Script |
| "Domain-Driven Design" (Evans 2003) | Domain Model 중심 사고 |
| "Hexagonal Architecture" (Alistair Cockburn 2005, blog) | Ports & Adapters 원전 |
| "Onion Architecture" (Jeffrey Palermo 2008) | 동심원 모델 (Hexagonal 의 변형) |
| "Clean Architecture" (Robert C. Martin 2017) | 4 동심원 + dependency rule |
| "Implementing Domain-Driven Design" (Vernon 2013) | Hexagonal + DDD 결합 |

---

## 10. 본 영역의 유지보수

| 트리거 | 갱신 항목 |
| --- | --- |
| 새 패턴 추가 | §2 인덱스 + 신규 노트 |
| 결정 매트릭스 변경 | §3 |
| 검증 도구 추가 | §8 |

---

## 11. 관련

- [[../patterns|↑ 40-patterns]]
- [[pattern-layered-architecture]]
- [[pattern-hexagonal-architecture]]
- [[pattern-clean-architecture]]
- [[../refactoring-patterns/refactoring-patterns|↗ refactoring-patterns hub]]
- [[../../20-areas/computer-science/object-oriented-programming/oop-for-backend|↗ OOP for backend]]
- [[../../20-areas/computer-science/object-oriented-programming/ddd-tactical-patterns|↗ DDD tactical patterns]]
- [[../../20-areas/computer-science/software-engineering/software-engineering|↗ software engineering]]
