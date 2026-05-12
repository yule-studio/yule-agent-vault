---
title: "Data Structure — 자료구조 사전형 인덱스"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T05:30:00+09:00
tags:
  - data-structure
  - index
---

# Data Structure — 자료구조 사전 hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 최초 — 9 sub-area + 학습 경로 |

**[[../computer-science|↑ computer-science]]** · **[[../../areas|↑↑ 20-areas]]**

---

## 1. 자료구조란?

데이터를 **저장하고 / 접근하고 / 수정하는 방법** 을 정의한 추상 모델 + 구현.
알고리즘의 효율은 데이터를 어떻게 담느냐로 결정 — **자료구조 + 알고리즘은
한 짝**.

> "프로그램 = 자료구조 + 알고리즘" — Niklaus Wirth (Pascal 창시자)

---

## 2. 9 영역 카탈로그

| 영역 | 진입점 | 핵심 / 사용처 |
| --- | --- | --- |
| 📋 **arrays-and-strings** | [[arrays-and-strings/arrays-and-strings]] | 가장 기본. 인덱스 O(1) / 삽입 O(N). 모든 알고리즘의 토대 |
| 🔗 **linked-lists** | [[linked-lists/linked-lists]] | 동적 크기 + O(1) 삽입. LRU 캐시 / 큐 / hash chaining |
| 📚 **stacks-and-queues** | [[stacks-and-queues/stacks-and-queues]] | LIFO / FIFO. DFS / BFS / 함수 호출 / undo |
| 🌳 **trees** | [[trees/trees]] | BST / AVL / Red-Black / B-tree. DB 인덱스 / 검색 |
| 🏔 **heaps** | [[heaps/heaps]] | 우선순위 큐. Dijkstra / heap sort / top-K |
| #️⃣ **hash-tables** | [[hash-tables/hash-tables]] | O(1) 평균 lookup. dict / set / Redis |
| 🕸 **graphs** | [[graphs/graphs]] | 인접 행렬 vs 리스트. SNS / 네트워크 / 의존성 |
| 🌲 **tries** | [[tries/tries]] | 문자열 prefix 트리. 자동완성 / IP 라우팅 |
| 🚀 **advanced** | [[advanced/advanced]] | Segment Tree / Fenwick / Bloom Filter / Skip List / Union-Find |

---

## 3. 시간 / 공간 복잡도 종합표 (Big-O Cheat Sheet)

### 평균 (Average)

| 자료구조 | Access | Search | Insert | Delete | 공간 |
| --- | --- | --- | --- | --- | --- |
| Array | O(1) | O(N) | O(N) | O(N) | O(N) |
| Dynamic Array | O(1) | O(N) | O(1) amortized | O(N) | O(N) |
| Linked List (singly) | O(N) | O(N) | O(1) at head | O(1) at head | O(N) |
| Stack | O(N) | O(N) | O(1) | O(1) | O(N) |
| Queue (deque) | O(N) | O(N) | O(1) | O(1) | O(N) |
| Hash Table | N/A | **O(1)** | **O(1)** | **O(1)** | O(N) |
| BST (balanced) | O(log N) | O(log N) | O(log N) | O(log N) | O(N) |
| BST (unbalanced) | O(N) | O(N) | O(N) | O(N) | O(N) |
| AVL / Red-Black | O(log N) | O(log N) | O(log N) | O(log N) | O(N) |
| B-Tree | O(log N) | O(log N) | O(log N) | O(log N) | O(N) |
| Heap (binary) | O(N) | O(N) | O(log N) | O(log N) | O(N) |
| Trie | O(K) | O(K) | O(K) | O(K) | O(N×K) |
| Skip List | O(log N) | O(log N) | O(log N) | O(log N) | O(N) |

### 최악 (Worst)

| 자료구조 | Access | Search | Insert | Delete |
| --- | --- | --- | --- | --- |
| Hash Table | N/A | O(N) | O(N) | O(N) (충돌 폭증) |
| Self-balancing BST | O(log N) | O(log N) | O(log N) | O(log N) |
| Unbalanced BST | O(N) | O(N) | O(N) | O(N) |

> 출처: [Big-O Cheat Sheet](https://www.bigocheatsheet.com/) — 시각화 권장

---

## 4. 자료구조 선택 가이드

### "어떤 자료구조를 쓸 것인가?"

| 요구 | 선택 |
| --- | --- |
| 빠른 인덱스 접근 | Array / Dynamic Array |
| 빠른 삽입·삭제 (앞/뒤) | Deque / Linked List |
| 빠른 검색 + 키 매핑 | Hash Table (O(1) 평균) |
| 정렬된 검색 | Balanced BST (TreeMap / std::map) — O(log N) |
| 최솟값 / 최댓값 즉시 | Heap (Priority Queue) |
| 문자열 prefix 매칭 | Trie |
| 연결 관계 | Graph |
| 그룹 합치기 / 같은 그룹? | Union-Find |
| 구간 합 / 구간 최댓값 | Segment Tree / Fenwick Tree |
| 빠른 "존재 여부" + 메모리 절약 | Bloom Filter (오답 가능) |
| 우선순위 + 정렬 동시 | TreeMap with multi-key |

---

## 5. 학습 경로 — 4 단계

### 1 단계 (1-2 주) — 코딩 테스트 기본
1. [[arrays-and-strings/arrays-and-strings]] — 인덱스 / 슬라이싱 / 투 포인터
2. [[stacks-and-queues/stacks-and-queues]] — 스택 / 큐 / deque
3. [[hash-tables/hash-tables]] — 딕셔너리 / 집합

### 2 단계 (2-3 주) — 코딩 테스트 중급
4. [[linked-lists/linked-lists]] — 단일 / 이중 / 환형
5. [[trees/trees]] — BST / 트리 순회
6. [[heaps/heaps]] — 우선순위 큐 / heapq
7. [[graphs/graphs]] — 인접 리스트 / 인접 행렬

### 3 단계 (1-2 주) — 면접 / 시스템 설계
8. [[trees/trees]] 의 균형 트리 (AVL / Red-Black / B+tree)
9. [[advanced/advanced]] 의 Union-Find / Segment Tree
10. [[tries/tries]] — 자동완성 / 라우팅

### 4 단계 (선택, 고급)
11. Skip List / Bloom Filter / LSM-tree / Treap

---

## 6. 실습 환경

각 노트는 **11 언어 구현** 포함:
- Python (`list`, `dict`, `set`, `deque`, `heapq`)
- Java (`ArrayList`, `HashMap`, `TreeMap`, `ArrayDeque`, `PriorityQueue`)
- C++ (`vector`, `unordered_map`, `map`, `deque`, `priority_queue`)
- C (직접 구현 — 메모리 할당)
- JavaScript (`Array`, `Map`, `Set`)
- TypeScript (위와 동일 + 타입)
- Kotlin (`MutableList`, `HashMap`, `TreeMap`, `ArrayDeque`, `PriorityQueue`)
- Swift (`Array`, `Dictionary`, `Set`)
- Go (`slice`, `map`, `container/list`, `container/heap`)
- Rust (`Vec`, `HashMap`, `BTreeMap`, `VecDeque`, `BinaryHeap`)
- C# (`List`, `Dictionary`, `SortedDictionary`, `Queue`, `PriorityQueue`)

### 실습 도구
- [VisuAlgo](https://visualgo.net/) — 자료구조 동작 시각화 (한국어)
- [Algorithm Visualizer](https://algorithm-visualizer.org/) — 코드 + 애니메이션
- [Big-O Cheat Sheet](https://www.bigocheatsheet.com/) — 복잡도 표
- [LeetCode Top 100](https://leetcode.com/studyplan/top-100-liked/) — 자료구조 별 문제

---

## 7. 노트 표준 14 섹션 (algorithm 과 동일)

1. 한 줄 정의
2. 언제 쓰는가
3. 핵심 직관 / 메모리 레이아웃
4. 동작 원리 (예시)
5. 복잡도 분석
6. 의사 코드
7. **11 언어 구현 (내장 + 직접 구현)**
8. 변형 / 응용
9. 함정 / 안티패턴
10. 실전 풀이 절차
11. 대표 문제 + 풀이
12. 연습 문제 (난이도별)
13. 학습 자료
14. 관련 hub

---

## 8. 운영 규칙

1. **사전이 목표** — 한 노트로 처음부터 끝까지.
2. **직접 구현 + 내장 사용 모두** — 면접 / 실무 둘 다.
3. **메모리 레이아웃** — 자료구조의 핵심. 그림 / ASCII 다이어그램.
4. **알고리즘 ↔ 자료구조 cross-link** — 예: BFS ↔ Queue, Dijkstra ↔ Heap.
5. 출처 + 날짜 명시.

---

## 9. 학습 자료 (전체)

### 책
- Introduction to Algorithms (CLRS) — Part III (Data Structures)
- Algorithms 4th (Sedgewick) — 자료구조 + 알고리즘 결합
- The Algorithm Design Manual (Skiena) — 실용 가이드
- Cracking the Coding Interview — 자료구조 면접 표준

### 강의
- [동빈나 — 자료구조](https://www.youtube.com/@dongbinna)
- [MIT 6.006 Introduction to Algorithms](https://ocw.mit.edu/courses/6-006-introduction-to-algorithms-spring-2020/)
- [Stanford CS166 Data Structures](https://web.stanford.edu/class/cs166/)

### 시각화
- [VisuAlgo](https://visualgo.net/en) — 자료구조 + 알고리즘 시각화
- [USFCA Visualization](https://www.cs.usfca.edu/~galles/visualization/Algorithms.html)

### 한국어
- "이것이 자료구조 + 알고리즘이다 with C 언어" (한빛미디어)
- 인프런 자료구조 강의 다수

---

## 10. 관련

- [[../algorithm/algorithm|↗ algorithm 사전]]
- [[../database-theory/database-theory|↗ DB 인덱스 (B+tree)]]
- [[../operating-system/operating-system|↗ OS 메모리 관리]]
- [[../../00-inbox/links/general/system-design|↗ system-design 링크]]
