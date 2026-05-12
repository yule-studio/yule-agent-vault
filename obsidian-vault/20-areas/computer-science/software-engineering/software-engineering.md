---
title: "소프트웨어 공학 (Software Engineering)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T12:00:00+09:00
tags:
  - software-engineering
  - clean-code
  - testing
  - solid
  - architecture
---

# 소프트웨어 공학 (Software Engineering)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 수준 |

**[[../computer-science|↑ computer-science]]**

---

## 1. 한 줄 정의

**작동하는 코드를 넘어 — 변경 가능 / 이해 가능 / 검증 가능한 시스템** 을 만드는
공학. Brooks 1975 "The Mythical Man-Month" 이후 본질이 바뀌지 않은 분야.

---

## 2. 핵심 원리

### 2.1 KISS (Keep It Simple, Stupid)
복잡함은 적. 가장 단순한 작동하는 해가 가장 좋다.

### 2.2 DRY (Don't Repeat Yourself)
지식의 중복 피하기. 단, **세 번** 반복까지 보고 추상화 (Rule of Three).

### 2.3 YAGNI (You Aren't Gonna Need It)
"미래에 쓸 것 같아서" 만들지 마라. 필요할 때 만든다.

### 2.4 SOLID (Robert C. Martin)

| 원칙 | 의미 |
| --- | --- |
| **S**ingle Responsibility | 한 클래스는 하나의 변경 이유 |
| **O**pen/Closed | 확장 OK, 수정 X |
| **L**iskov Substitution | 자식은 부모를 대체 가능 |
| **I**nterface Segregation | 작은 인터페이스 여러 개 |
| **D**ependency Inversion | 추상에 의존, 구체에 X |

### 2.5 GRASP (Larman)
- High Cohesion
- Low Coupling
- Information Expert
- Creator / Controller / Pure Fabrication

### 2.6 Law of Demeter
"친구하고만 대화" — `a.b.c.d` 같은 chain 피하기.

### 2.7 Tell, Don't Ask
객체에게 일을 시키지, 상태를 묻고 결정하지 마라.

---

## 3. Clean Code (Martin)

### 3.1 의미 있는 이름

```python
# Bad
d = 30  # days

# Good
days_until_expiration = 30
```

### 3.2 함수

- 작게 — 한 화면 안에
- 한 가지만 — 한 추상 수준
- 인수 적게 (0, 1, 2 이상은 의심)
- 부수 효과 없게
- 명령 / 질의 분리 (CQS)

### 3.3 주석

- 좋은 주석: 의도, 경고, TODO, 법적
- 나쁜 주석: 코드 설명, 마커, 닫는 괄호 라벨, 죽은 코드

### 3.4 형식 / 구성

- 신문 비유 — 위는 추상, 아래는 상세
- 수직 거리 — 관련 함수는 가까이
- 변수는 사용 직전에

---

## 4. 테스트

### 4.1 테스트 피라미드 (Mike Cohn)

```
       [E2E]      ← 적음 (느림, 비쌈)
     [Integration]
   [   Unit    ]   ← 많음 (빠름, 쌈)
```

### 4.2 단위 테스트 (FIRST)

- **F** ast
- **I** ndependent
- **R** epeatable
- **S** elf-validating
- **T** imely

### 4.3 TDD (Test-Driven Development)

```
Red → Green → Refactor
실패 테스트 → 통과 → 개선
```

Kent Beck "Test-Driven Development: By Example".

### 4.4 BDD (Behavior-Driven)

```
Given (조건)
When (행동)
Then (결과)
```

Cucumber, RSpec.

### 4.5 테스트 더블 (Gerard Meszaros)

| 종류 | 설명 |
| --- | --- |
| **Dummy** | 인자만 채움 |
| **Fake** | 작동하지만 단순 (in-memory DB) |
| **Stub** | 미리 정해진 답 |
| **Spy** | Stub + 호출 기록 |
| **Mock** | 기대값 검증 |

### 4.6 Property-Based Testing

- 임의 입력 자동 생성
- QuickCheck (Haskell), Hypothesis (Python)
- 불변량 검증

### 4.7 변이 테스트 (Mutation Testing)

- 코드를 의도적으로 망가뜨려 테스트가 잡는지
- 테스트의 테스트 — PIT, mutmut

### 4.8 Coverage

- Line / Branch / Path coverage
- 100% 가 목표 아님 — 위험한 곳 우선

---

## 5. 아키텍처 패턴

### 5.1 레이어드 (N-Tier)

```
Presentation (UI)
   ↓
Application (Use case)
   ↓
Domain (Business)
   ↓
Infrastructure (DB / API)
```

### 5.2 헥사고날 (Hexagonal / Ports & Adapters)

- 도메인 중심
- 외부 (DB, UI, API) 는 Adapter 로
- 의존성 방향이 도메인을 향함

### 5.3 Clean Architecture (Uncle Bob)

```
Entities ← Use Cases ← Interface Adapters ← Frameworks/Drivers
```

### 5.4 DDD (Domain-Driven Design, Evans 2003)

- **Bounded Context** — 모델의 경계
- **Ubiquitous Language** — 도메인 전문가와 공통 어휘
- **Aggregate** — 일관성 경계
- **Entity / Value Object**
- **Domain Event**
- **Repository**

### 5.5 CQRS / Event Sourcing

[[../distributed-systems/distributed-systems#10 분산 패턴]] 참조.

### 5.6 MVC / MVP / MVVM

| 패턴 | 출현 |
| --- | --- |
| MVC | 1979, Smalltalk |
| MVP | 1990s, Taligent |
| MVVM | 2005, Microsoft WPF |

---

## 6. 리팩토링 (Fowler)

### 6.1 코드 냄새 (Code Smells)

- **중복** — Extract Method
- **긴 함수** — Extract Method
- **큰 클래스** — Extract Class
- **긴 인자 목록** — Introduce Parameter Object
- **Feature Envy** — 다른 클래스 메서드 자주 호출 → Move Method
- **Data Clump** — 같이 다니는 데이터 → Extract Class
- **Primitive Obsession** — int / string 남용 → Value Object
- **Switch Statements** — Replace with Polymorphism
- **Shotgun Surgery** — 한 변경이 여러 곳 → Move Method/Field

### 6.2 핵심 리팩토링

- Extract Function
- Inline Function
- Extract Variable
- Move Function
- Rename
- Replace Conditional with Polymorphism
- Replace Magic Number with Constant

Fowler "Refactoring" 2판 (2018) 권위서.

---

## 7. 버전 관리 (Git)

### 7.1 핵심 개념

- Commit, Branch, Merge, Rebase, Tag
- Remote, Origin, Upstream
- Reflog, Cherry-pick, Stash

### 7.2 워크플로우

- **Git Flow** — develop / feature / release / hotfix (Vincent Driessen 2010)
- **GitHub Flow** — main + feature branches (단순)
- **GitLab Flow** — environment branches
- **Trunk-Based** — short-lived branch + feature flag

### 7.3 커밋 메시지 컨벤션

- **Conventional Commits** — `feat: add login`, `fix: handle null`
- 본문에 "왜" 설명
- 50/72 룰 (제목 50 자, 본문 72 자 줄바꿈)

### 7.4 PR / Code Review

- 작게, 자주
- 자기 설명 코드 + PR 본문에 맥락
- 리뷰어는 작성자보다 후순위

---

## 8. CI/CD

### 8.1 CI (Continuous Integration)

- 매 커밋마다 빌드 + 테스트
- 빠른 피드백
- Jenkins, GitHub Actions, GitLab CI, CircleCI

### 8.2 CD (Continuous Delivery / Deployment)

- **Delivery** — 언제든 배포 가능 (수동 트리거)
- **Deployment** — 자동 배포

### 8.3 배포 전략

- **Blue-Green** — 두 환경 교체
- **Canary** — 일부 트래픽
- **Rolling** — 점진적 교체
- **Feature Flag** — 런타임 토글
- **Shadow** — 트래픽 복제, 응답 무시

---

## 9. 추정 / 일정

### 9.1 Brooks's Law

> "지연되는 프로젝트에 사람을 더 투입하면 더 늦어진다"

### 9.2 No Silver Bullet (Brooks 1986)

- 본질 (Essential) vs 우연 (Accidental) 복잡도
- 어떤 기술도 10x 생산성 증가 X

### 9.3 추정 기법

- **Planning Poker** — Fibonacci
- **T-shirt sizing** — S/M/L/XL
- **3-point estimate** — Best / Worst / Most likely

### 9.4 Hofstadter's Law

> "예상보다 항상 더 걸린다 — Hofstadter's Law 를 고려해도"

---

## 10. 함정 / 안티패턴

### 함정 1 — 과도한 추상화
미래 가정으로 인터페이스 / 팩토리 폭발. YAGNI.

### 함정 2 — God Object
한 클래스에 모든 책임. SRP 위반.

### 함정 3 — 깨진 창문 이론
방치된 나쁜 코드 → 더 나쁜 코드 유발.

### 함정 4 — 실버 불릿 신앙
"이 프레임워크가 다 해결" — 항상 거짓.

### 함정 5 — 너무 많은 주석
이름이 잘못된 코드의 보상.

### 함정 6 — 테스트 없이 리팩토링
넷이 깨질지 모름. 테스트 먼저.

### 함정 7 — Boy Scout Rule 의 오용
"왔을 때보다 깨끗하게" — PR 의 범위를 벗어남.

### 함정 8 — 마지막 책임 미루기 (Last Responsible Moment) 의 오용
결정 미루기 ≠ 결정 회피.

---

## 11. 메트릭

### 11.1 코드 메트릭

- **Cyclomatic Complexity** — 분기 수 (10 이상 위험)
- **Cognitive Complexity** — 인간이 이해하기 어려움 정도
- **LOC** — 의미 적음
- **Coupling / Cohesion**
- **Test Coverage** — 위에서 봄
- **Tech Debt Ratio**

### 11.2 DORA 메트릭

- **Deployment Frequency**
- **Lead Time for Changes**
- **MTTR** (Mean Time to Recovery)
- **Change Failure Rate**

### 11.3 SPACE Framework

- Satisfaction, Performance, Activity, Communication, Efficiency

---

## 12. 개발자 경험 (DX)

### 12.1 좋은 DX 의 요소

- 빠른 로컬 빌드
- 신뢰할 수 있는 테스트
- 명확한 에러 메시지
- 좋은 문서
- 자동화

### 12.2 Documentation

- **README** — 첫 인상
- **Architecture Decision Records (ADR)** — 결정의 이유
- **Runbook** — 운영 절차
- **Postmortem** — 장애 회고 (blameless)

---

## 13. 학습 자료

- **Clean Code** (Martin) — 한국어로 번역됨
- **Refactoring** 2판 (Fowler 2018)
- **The Pragmatic Programmer** (Hunt / Thomas) — 20주년판
- **Mythical Man-Month** (Brooks)
- **A Philosophy of Software Design** (Ousterhout) — 깊이의 모듈
- **Refactoring to Patterns** (Joshua Kerievsky)
- **Working Effectively with Legacy Code** (Feathers)
- **Test-Driven Development by Example** (Kent Beck)
- **DDD** (Eric Evans 2003) — 원전
- **Implementing Domain-Driven Design** (Vaughn Vernon)

---

## 14. 면접 질문

1. **SOLID 각각 설명**.
2. **테스트 피라미드와 의미**.
3. **TDD / BDD 차이**.
4. **DDD 의 Bounded Context**.
5. **DI / IoC 의 차이**.
6. **모놀리식 → 마이크로서비스 전환**.
7. **Code Smell 5 가지**.
8. **CI vs CD vs CD**.
9. **Brooks's Law / Mythical Man-Month**.
10. **DORA 메트릭**.

---

## 15. 관련

- [[../design-patterns/design-patterns]] — GoF 등 구체 패턴
- [[../distributed-systems/distributed-systems]] — 아키텍처
- [[../../soft-skills/soft-skills|↑ soft-skills]] — 팀워크
- [[../computer-science|↑ computer-science]]
