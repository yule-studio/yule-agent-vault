---
title: "refactoring-patterns — Hub"
kind: knowledge
project: patterns
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-18T23:30:00+09:00
tags: [patterns, refactoring, hub]
home_hub: patterns
related:
  - "[[../patterns]]"
  - "[[pattern-replace-conditional-with-polymorphism]]"
  - "[[../../20-areas/computer-science/object-oriented-programming/anti-patterns]]"
  - "[[../../20-areas/computer-science/object-oriented-programming/good-vs-bad-oop]]"
---

# refactoring-patterns — Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-18 | engineering-agent/tech-lead | 최초 — 카테고리 hub |

**[[../patterns|↑ 40-patterns]]**

---

## 1. 영역 정의

본 영역은 외부 동작을 보존하면서 내부 구조를 개선하는 **리팩토링 패턴** 의 카탈로그다. 단순 코드 냄새의 1:1 fix 가 아니라 **여러 단계에 걸친 큰 단위 리팩토링** 을 절차로 정리한다.

본 영역이 정의하는 것:
- 복잡한 리팩토링의 단계별 절차
- 각 리팩토링의 적용 조건 / 비용 / 회피 시점
- 테스트 / CI 와 결합된 안전한 변경 절차
- legacy 코드의 점진적 개선 전략

본 영역이 정의하지 않는 것:
- Fowler 의 작은 단위 리팩토링 카탈로그 (Extract Method / Rename / Inline 등) — Fowler "Refactoring" 2판 참고
- 안티패턴 정의 자체 — [[../../20-areas/computer-science/object-oriented-programming/anti-patterns]]
- 도메인 설계 패턴 — [[../architecture-patterns/architecture-patterns]]

---

## 2. 패턴 인덱스

| 패턴 | 노트 | 한 줄 정의 |
| --- | --- | --- |
| Replace Conditional with Polymorphism | [[pattern-replace-conditional-with-polymorphism]] | switch / instanceof / string type discriminator → polymorphism |

→ 본 영역은 점진 추가. 다음 후보:
- Strangler Fig (legacy → new 점진 교체)
- Branch by Abstraction (큰 리팩토링 중에도 trunk 유지)
- Extract Class (god class 분해)
- Move Method (책임 재배치)
- Replace Inheritance with Composition

---

## 3. Fowler "Refactoring" 카탈로그와의 관계

Fowler 의 2 판 (2018) 은 **작은 단위 리팩토링 60+ 개** 의 카탈로그. 본 영역은:

| 영역 | 다루는 것 |
| --- | --- |
| Fowler 카탈로그 | Extract Method / Rename / Inline / Move Field / 같은 작은 단위. step 1-3. |
| 본 영역 (refactoring-patterns) | 큰 단위 리팩토링 — 여러 작은 단위의 조합 + 전략. step 5-20. |

→ 본 영역의 각 패턴은 **여러 Fowler 카탈로그를 묶은 절차** 이며 production 안전성 (테스트 / CI / 기능 플래그) 까지 다룬다.

---

## 4. 적용 절차 — 공통

모든 리팩토링은 다음 절차를 따른다.

| 단계 | 작업 |
| --- | --- |
| 1 | 리팩토링 대상 식별 (안티패턴 / 코드 냄새 / metric) |
| 2 | 외부 동작 검증할 테스트 작성 / 보강 (없으면 characterization test) |
| 3 | 작은 step 으로 분해 (한 step = 한 Fowler 리팩토링) |
| 4 | 각 step 후 컴파일 + 테스트 통과 확인 |
| 5 | 작은 PR 로 분할 — 한 PR 에 한 리팩토링 패턴 |
| 6 | metric 비교 (LCOM / cyclomatic / coupling) 로 효과 검증 |

---

## 5. 리팩토링과 비기능 변경의 분리

| 변경 종류 | PR 분리 |
| --- | --- |
| 순수 리팩토링 (외부 동작 0) | 단독 PR |
| 비즈니스 로직 변경 | 단독 PR |
| 리팩토링 + 비즈니스 로직 동시 | 분리 권장 — review 어려움 |

→ "리팩토링" 과 "기능 추가" 를 같은 PR 에 섞으면 review 시 무엇이 변경되었는지 추적 어려움. 별도 commit / PR 권장.

---

## 6. 안전 장치

| 장치 | 효과 |
| --- | --- |
| **테스트 (unit / integration / e2e)** | 외부 동작 보존 검증 |
| **CI 자동 실행** | 매 commit 검증 |
| **feature flag** | 새 코드 경로를 안전하게 활성/비활성 |
| **branch by abstraction** | 큰 리팩토링 중에도 trunk 에 머무름 |
| **strangler fig** | 새 코드가 점진적으로 옛 코드 대체 |
| **canary / blue-green** | 일부 트래픽만 새 코드로 |
| **rollback plan** | 문제 시 즉시 복원 |

---

## 7. 적용 조건

| 조건 | 의미 |
| --- | --- |
| 테스트 커버리지 충분 | 외부 동작 검증 가능 |
| code review 문화 | 리뷰어가 안전성 검증 |
| 작은 PR 가능 | 단계별 검증 |
| CI 빠름 | 빠른 피드백 |
| rollback 쉬움 | 위험 감수 가능 |

조건 미충족 시 리팩토링 우선 X — 먼저 테스트 / CI / 작은 PR 문화 확립.

---

## 8. 측정 — 리팩토링의 효과

리팩토링 전후 다음 지표로 효과 검증.

| 지표 | 측정 |
| --- | --- |
| LCOM4 (응집) ↓ | 클래스 응집 ↑ |
| fan-out (결합) ↓ | 외부 의존 감소 |
| cyclomatic / cognitive complexity ↓ | 분기 단순화 |
| PR 평균 파일 수 ↓ | shotgun surgery 감소 |
| test coverage ↑ | 안전한 변경 영역 확대 |
| bug fix lead time ↓ | 영향 범위 축소 |

상세: [[../../20-areas/computer-science/object-oriented-programming/coupling-cohesion]] §9.

---

## 9. 외부 표준 / 참고

| 자료 | 영역 |
| --- | --- |
| "Refactoring" 2판 (Fowler 2018) | 60+ 작은 단위 카탈로그 |
| "Refactoring to Patterns" (Kerievsky 2004) | 안티패턴 → GoF 패턴 변환 |
| "Working Effectively with Legacy Code" (Feathers 2004) | seam / characterization test / legacy 점진 개선 |
| "Practical Object-Oriented Design" (Sandi Metz 2018) | small step refactoring |
| "Continuous Delivery" (Humble / Farley 2010) | branch by abstraction / strangler fig |

---

## 10. 본 영역의 유지보수

| 트리거 | 갱신 항목 |
| --- | --- |
| 새 패턴 추가 | §2 인덱스 + 신규 노트 |
| 적용 조건 변경 | §7 |
| metric 추가 | §8 |

---

## 11. 관련

- [[../patterns|↑ 40-patterns]]
- [[pattern-replace-conditional-with-polymorphism]]
- [[../architecture-patterns/architecture-patterns|↗ architecture-patterns hub]]
- [[../../20-areas/computer-science/object-oriented-programming/anti-patterns|↗ OOP anti-patterns]]
- [[../../20-areas/computer-science/object-oriented-programming/good-vs-bad-oop|↗ 좋은 OOP vs 나쁜 OOP]]
- [[../../20-areas/computer-science/object-oriented-programming/coupling-cohesion|↗ coupling / cohesion 측정]]
