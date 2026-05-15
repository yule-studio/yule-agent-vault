# 40-patterns/ — 재사용 설계 패턴

> 한 번 정리하면 여러 프로젝트에서 다시 쓰는 **설계 / 구현 패턴**.
> 단순 코드 조각 (50-snippets) 과 달리, 사용 맥락 + 트레이드오프 +
> 안티패턴을 함께 담는다.

**[[../index|↑ vault 인덱스]]**

## 카테고리

| 카테고리 | 진입점 | 예시 |
| --- | --- | --- |
| architecture-patterns | [[architecture-patterns/]] | 마이크로서비스 / 헥사고날 / CQRS |
| api-patterns | [[api-patterns/]] | REST / GraphQL / gRPC / pagination |
| design-patterns | [[design-patterns/]] | 전략 / 옵저버 / 팩토리 (GoF) |
| deployment-patterns | [[deployment-patterns/]] | Blue-Green / Canary / Rolling |
| error-handling-patterns | [[error-handling-patterns/]] | Circuit Breaker / Retry / Bulkhead |
| refactoring-patterns | [[refactoring-patterns/]] | Strangler Fig / Branch by Abstraction |
| github-conventions | [[github-conventions/github-conventions]] | 이슈 6 종 / PR 템플릿 / 라벨 priority |
| git-conflict | [[git-conflict/git-conflict]] | 백업 / 머지용·해결용 브랜치 분리 / 단계별 해결 / 예방 |
| documentation-versioning | [[documentation-versioning]] | 문서 버전 (semver) — Major/Minor/Patch bump 규칙 |

## 파일명 컨벤션

```
pattern-<name-kebab>.md
```

예: `pattern-circuit-breaker.md`, `pattern-saga.md`.

## 노트 표준 구조

```markdown
# <패턴 이름>

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | YYYY-MM-DD | <역할> | 최초 정리 |

## 의도
이 패턴이 풀려는 문제 (1 문장).

## 동기 (Motivation)
왜 이 패턴이 필요한 상황이 생기는가.

## 구조 / 다이어그램
패턴의 핵심 구성 요소 (텍스트 다이어그램 또는 mermaid).

## 적용 예
실제 코드 또는 시스템 예 (구체적).

## 트레이드오프
- 장점
- 단점
- 언제 쓰면 안 되는가

## 안티패턴
잘못 적용한 모습.

## 관련
- [[20-areas/.../concept-X]] (이 패턴이 의존하는 개념)
- [[30-resources/.../book-Y]] (출처)
- [[10-projects/.../decision-Z]] (이 패턴을 채택한 결정)
```

## 운영 규칙

1. **재사용 가능성** 이 핵심. 한 프로젝트 한정이면 `10-projects/` 안에.
2. 패턴은 **이름이 합의된 것** 만 (책 / 컨퍼런스 / 업계 통용 명칭).
   새 이름 짓는 건 위험.
3. 추상 설명 + 구체 예 양쪽 모두 필수.
4. 같은 카테고리의 다른 패턴과 [[cross-link]] (예: Circuit Breaker ↔
   Retry ↔ Bulkhead).

## 관련

- [[../index|↑ vault 인덱스]]
- [[../20-areas/README|20-areas]] — 개념 정리
- [[../50-snippets/README|50-snippets]] — 짧은 코드 조각
- [[../30-resources/README|30-resources]] — 패턴의 출처 자료
