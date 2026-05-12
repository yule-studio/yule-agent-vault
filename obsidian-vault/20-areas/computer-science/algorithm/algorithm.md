---
title: "Algorithm — 사전형 인덱스 (이코테 기반)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T00:00:00+09:00
tags:
  - algorithm
  - computer-science
  - coding-test
  - index
---

# Algorithm — 코딩 테스트 / 알고리즘 사전

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 최초 — 이코테 (나동빈) 목차 기반 + 4 언어 구현 |

> "이것이 취업을 위한 코딩 테스트다" (나동빈, 한빛미디어) 목차를 그대로
> 따라 정리한 **알고리즘 개념 사전**. 각 노트는 **개념 → 시간/공간 복잡도 →
> 의사 코드 → Python / Java / C++ / C 구현 → 대표 문제 → 안티패턴 → 학습
> 자료** 9 섹션 구조.

**[[../computer-science/computer-science|↑ computer-science]]** · **[[../../areas|↑↑ 20-areas]]**

## PART 01 — 코딩 테스트, 무엇을 어떻게 준비할까?

- [[intro/coding-test-overview]] — 코딩 테스트 개념 / 배경 / 실습 환경
- [[intro/complexity]] — 시간 복잡도 / 공간 복잡도 / Big-O
- [[intro/interview-process]] — 채용 프로세스 / 기술 면접 유형 / 준비법
- [[intro/problem-solving-sites]] — 알고리즘 문제 풀이 사이트 / 커뮤니티

## PART 02 — 주요 알고리즘 이론

| 챕터 | 진입점 | 핵심 키워드 |
| --- | --- | --- |
| Chapter 03 | [[greedy/greedy]] | 그리디 / 당장 좋은 것 / 정당성 증명 |
| Chapter 04 | [[implementation/implementation]] | 구현 / 시뮬레이션 / 완전 탐색 |
| Chapter 05 | [[dfs-bfs/dfs-bfs]] | DFS / BFS / 스택 / 큐 / 재귀 |
| Chapter 06 | [[sorting/sorting]] | 선택 / 삽입 / 퀵 / 병합 / 계수 정렬 |
| Chapter 07 | [[binary-search/binary-search]] | 이진 탐색 / 파라메트릭 서치 |
| Chapter 08 | [[dynamic-programming/dynamic-programming]] | DP / 메모이제이션 / 타뷸레이션 |
| Chapter 09 | [[shortest-path/shortest-path]] | 다익스트라 / 플로이드–워셜 / 벨만–포드 |
| Chapter 10 | [[graph-theory/graph-theory]] | 서로소 집합 / MST (크루스칼/프림) / 위상 정렬 |

## PART 03 — 알고리즘 유형별 기출문제

PART 02 의 각 챕터 안 `## 대표 문제` 섹션 + `## 기출` 섹션에서 문제 풀이를
한 곳에서 본다. 별도 폴더 만들지 않고 알고리즘 노트 안에 통합.

## PART 04 — 부록

- [[reference/python-syntax-for-ct]] — 코딩 테스트용 Python 문법
- [[reference/stl-cheatsheet]] — Python / Java Collection / C++ STL / C 표준 라이브러리
- [[reference/additional-algorithms]] — 기타 알고리즘 (트라이 / 세그먼트 트리 / KMP / 등)

## 노트 표준 구조 (9 섹션)

```markdown
# <알고리즘 이름>

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | YYYY-MM-DD | <역할> | 최초 |

## 1. 개요
한 문장으로 "무엇인가".

## 2. 핵심 개념
- 사용 상황 / 핵심 아이디어
- 다른 알고리즘과의 차이

## 3. 동작 원리
단계 1 → 단계 2 → ... (예시 데이터로)

## 4. 복잡도
- 시간: O(...)
- 공간: O(...)

## 5. 의사 코드 (pseudo)
```
function ALGO(input):
    ...
```

## 6. 언어별 구현 (코테 가능 11 언어)
### Python (Python 3)
### Java
### C++
### C
### JavaScript (Node.js)
### TypeScript
### Kotlin
### Swift
### Go
### Rust
### C#

## 7. 변형 / 응용
- 비슷한 문제 유형 / 변형

## 8. 대표 문제
- (이코테 실전 문제 + 솔루션)

## 9. 안티패턴 / 함정
- 자주 틀리는 부분

## 10. 학습 자료
- 책 / 강의 / 사이트
```

## 운영 규칙

1. **사전** 이 목표 — 검색해서 한 노트만 봐도 그 알고리즘을 처음부터
   끝까지 이해할 수 있어야 함.
2. **4 언어 구현** — Python (이코테 기본) / Java / C++ / C 모두 동작
   가능한 코드. 주석으로 핵심 단계 표시.
3. 책의 실전 문제 / 기출 문제는 출처 (이코테 ch.NN) 명시 + 풀이 코드.
4. 같은 알고리즘의 다른 응용은 한 노트 안 `## 변형` 섹션. 노트 분리하지
   않음.
5. 알고리즘 변형이 별도 알고리즘 (예: 최단 경로 안 다익스트라 vs 벨만-
   포드) 이면 서브 폴더 안 별도 노트 + 폴더 인덱스에서 묶음.

## 관련

- [[../computer-science|↑ computer-science 영역]]
- [[../../../00-inbox/links/general/system-design|↗ system-design 링크]]
- [[../../../00-inbox/links/engineering/backend/java|↗ Java reference]]
- [[../../../00-inbox/links/engineering/ai/llm-foundations|↗ LLM foundations]]

## 출처

- "이것이 취업을 위한 코딩 테스트다" (나동빈, 한빛미디어, 2020)
- [동빈나 YouTube](https://www.youtube.com/@dongbinna)
- [GitHub: ndb796/python-for-coding-test](https://github.com/ndb796/python-for-coding-test)
