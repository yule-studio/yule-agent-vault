---
title: "computer-science — CS 지식 사전형 인덱스"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T05:00:00+09:00
tags:
  - computer-science
  - index
  - area
---

# computer-science — CS 사전 hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.2.0.0 | 2026-05-13 | engineering-agent/tech-lead | 13 영역 확장 + algorithm 사전 완결 반영 + 학습 경로 |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 최초 — 6 영역 |

> 컴퓨터 과학 (Computer Science) 의 핵심 영역을 묶은 학습 hub. 알고리즘 /
> 자료 구조 / 데이터베이스 이론 / 네트워크 / 운영체제 / 소프트웨어 공학 /
> 분산 시스템 / 보안 이론 / 디자인 패턴 / 컴파일러 / 하드웨어 / 컴퓨터 구조.

**[[../areas|↑ 20-areas]]**

---

## 13 영역 — 학습 우선순위 표

### Tier 1 — 모든 개발자 필수 (코딩 테스트 / 신입 면접)

| 영역 | 진입점 | 상태 |
| --- | --- | --- |
| 🧮 **algorithm** | [[algorithm/algorithm]] | ✅ **사전 완성** — 이코테 8 챕터 + 11 언어 |
| 📦 data-structure | [[data-structure/data-structure]] | 🚧 작성 중 (배열/리스트/스택큐/트리/해시/힙) |
| 🌐 network | [[network/network]] | 📝 placeholder |
| 💻 operating-system | [[operating-system/operating-system]] | 📝 placeholder |
| 🗄 database-theory | [[database-theory/database-theory]] | 📝 placeholder |

### Tier 2 — 중급 / 시스템 면접

| 영역 | 진입점 | 상태 |
| --- | --- | --- |
| 🏗 software-engineering | [[software-engineering/software-engineering]] | 📝 placeholder |
| 🎨 design-patterns | [[design-patterns/design-patterns]] | 📝 placeholder — GoF 23 + 아키텍처 |
| 🌐 distributed-systems | [[distributed-systems/distributed-systems]] | 📝 placeholder — CAP / 합의 |

### Tier 3 — 고급 / 시니어 면접

| 영역 | 진입점 | 상태 |
| --- | --- | --- |
| 🏛 computer-architecture | [[computer-architecture/computer-architecture]] | 📝 placeholder — CPU / 캐시 / ISA |
| 🔐 security-theory | [[security-theory/security-theory]] | 📝 placeholder — 암호학 / 인증 |
| 🛠 compiler | [[compiler/compiler]] | 📝 placeholder — 어휘 / 구문 / 의미 |
| ⚙️ hardware | [[hardware/hardware]] | 📝 placeholder — 메모리 / 저장 / SoC |
| 🗄 database (기존) | [[database/]] | 레거시 보존 |

---

## 학습 경로 — 8 단계 로드맵

### 1 단계 (1-3 개월) — 코딩 테스트 통과
1. [[algorithm/algorithm]] 의 PART 02 (8 챕터) 정독 — 이미 사전 완성
2. [[data-structure/data-structure]] 기초 — 배열 / 리스트 / 스택 큐 / 해시 / 트리 / 힙
3. 백준 단계별 → 실버 ~ 골드 5

### 2 단계 (1-2 개월) — 시스템 기초
4. [[operating-system/operating-system]] — 프로세스 / 스레드 / 메모리
5. [[network/network]] — TCP/IP / HTTP / TLS
6. [[database-theory/database-theory]] — 인덱스 / 트랜잭션 / 정규화

### 3 단계 (1-2 개월) — 소프트웨어 공학
7. [[software-engineering/software-engineering]] — 테스트 / 리팩터링 / SDLC
8. [[design-patterns/design-patterns]] — GoF + 아키텍처 패턴

### 4 단계 (2-3 개월) — 분산 / 시니어 면접
9. [[distributed-systems/distributed-systems]] — CAP / Raft / DDIA 책
10. 시스템 설계 면접 — [[../../00-inbox/links/general/system-design|↗ system-design 링크]]

### 5 단계 (선택) — 깊이
11. [[computer-architecture/computer-architecture]]
12. [[security-theory/security-theory]]
13. [[compiler/compiler]]
14. [[hardware/hardware]]

---

## 사전 수준 노트 표준 14 섹션

algorithm 의 모든 노트가 따르는 표준 (~800-1000 줄):

1. **한 줄 정의**
2. **언제 쓰는가** (시그널 인식 + 다른 알고리즘과 비교)
3. **핵심 직관** (비유 / 그림)
4. **동작 원리** (예시 데이터로 깊이)
5. **복잡도 분석**
6. **의사 코드**
7. **언어별 구현 11 개** — Python / Java / C++ / C / JavaScript / TypeScript / Kotlin / Swift / Go / Rust / C#
8. **변형 / 응용**
9. **함정 / 안티패턴**
10. **실전 풀이 절차**
11. **대표 문제 + 풀이 코드**
12. **연습 문제 (난이도별)**
13. **학습 자료** (책 / 강의 / 시각화)
14. **관련 hub**

다른 CS 영역도 같은 구조로 작성:
- 알고리즘 → 동작 원리 + 구현
- 자료구조 → 메모리 레이아웃 + 연산 + 구현
- OS → 메커니즘 + 시스템 콜 / 실습
- 네트워크 → 프로토콜 + 패킷 + 실습 (Wireshark / curl)
- DB 이론 → SQL / 인덱스 / 트랜잭션 + EXPLAIN 실습

---

## 운영 규칙

1. **사전이 목표** — 한 노트로 그 개념을 처음부터 끝까지 이해 가능해야.
2. **실습 가능** — 모든 노트에 "직접 따라할 코드 / 명령어".
3. **11 언어 구현** — 알고리즘처럼. 자료구조 / OS / 네트워크 모두 동일.
4. **단순 복붙 금지** — 내 해석 / 정리 / 적용 예 50% 이상.
5. **출처 + 날짜 필수**.
6. **CS 이론 ↔ 실무 cross-link** — `database-theory/transaction` ↔ `database/postgresql/transaction-tuning`.

---

## algorithm 완성 보고

이코테 (이것이 취업을 위한 코딩 테스트다, 나동빈) PART 02 전체:

| Ch | 영역 | 노트 | 줄 수 |
| --- | --- | --- | --- |
| 03/11 | 그리디 | [[algorithm/greedy/greedy]] | ~800 |
| 04/12 | 구현 | [[algorithm/implementation/implementation]] | ~900 |
| 05/13 | DFS/BFS | [[algorithm/dfs-bfs/dfs-bfs]] | ~950 |
| 06/14 | 정렬 | [[algorithm/sorting/sorting]] | ~770 |
| 07/15 | 이진 탐색 | [[algorithm/binary-search/binary-search]] | ~835 |
| 08/16 | DP | [[algorithm/dynamic-programming/dynamic-programming]] | ~760 |
| 09/17 | 최단 경로 | [[algorithm/shortest-path/shortest-path]] | ~760 |
| 10/18 | 그래프 이론 | [[algorithm/graph-theory/graph-theory]] | ~915 |
| — | 메인 인덱스 | [[algorithm/algorithm]] | ~130 |
| — | 코딩 테스트 개요 | [[algorithm/intro/coding-test-overview]] | — |
| — | 복잡도 | [[algorithm/intro/complexity]] | — |
| — | 면접 프로세스 | [[algorithm/intro/interview-process]] | — |
| — | 문제 풀이 사이트 | [[algorithm/intro/problem-solving-sites]] | — |

**전체: 약 7000+ 줄 / 11 언어 / 50+ 대표 문제 풀이 / 100+ 연습 문제 링크.**

---

## 다음 진행

1. **data-structure** — 가장 우선 (algorithm 의 짝)
2. **operating-system / network / database-theory** — Tier 1 시스템 기초
3. **design-patterns / software-engineering** — Tier 2 소프트웨어 공학
4. 나머지 — 사용자 요청에 따라

---

## 관련

- [[../areas|↑ 20-areas]]
- [[../../00-inbox/links/general/system-design|↗ system-design 링크 카탈로그]]
- [[../../00-inbox/links/general/books|↗ CS 책 카탈로그]]
- [[../../00-inbox/links/engineering/backend/database/database|↗ database 실무 링크]]
