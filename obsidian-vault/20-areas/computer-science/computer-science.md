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
| v.3.0.0 | 2026-05-13 | engineering-agent/tech-lead | ✅ 13 영역 전체 사전 수준 본문화 완료 |
| v.2.0.0 | 2026-05-13 | engineering-agent/tech-lead | 13 영역 확장 + algorithm 사전 완결 반영 |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 최초 — 6 영역 |

> 컴퓨터 과학 (Computer Science) 의 핵심 영역을 묶은 학습 hub. 알고리즘 /
> 자료 구조 / 데이터베이스 이론 / 네트워크 / 운영체제 / 소프트웨어 공학 /
> 분산 시스템 / 보안 이론 / 디자인 패턴 / 컴파일러 / 하드웨어 / 컴퓨터 구조.

**[[../areas|↑ 20-areas]]**

---

## 🎉 13 영역 — 전체 사전 수준 완성

### Tier 1 — 모든 개발자 필수 (코딩 테스트 / 신입 면접)

| 영역 | 진입점 | 상태 |
| --- | --- | --- |
| 🧮 **algorithm** | [[algorithm/algorithm]] | ✅ **사전 완성** — 이코테 8 챕터 + 11 언어 |
| 📦 **data-structure** | [[data-structure/data-structure]] | ✅ **사전 완성** — 9 영역 (배열/리스트/스택큐/트리/힙/해시/그래프/트라이/고급) |
| 🌐 **network** | [[network/network]] | ✅ **사전 완성** — TCP/IP + HTTP + TLS + DNS |
| 💻 **operating-system** | [[operating-system/operating-system]] | ✅ **사전 완성** — Process/Thread/Memory/FS/IO |
| 🗄 **database-theory** | [[database-theory/database-theory]] | ✅ **사전 완성** — Codd/ACID/Index/CAP |

### Tier 2 — 중급 / 시스템 면접

| 영역 | 진입점 | 상태 |
| --- | --- | --- |
| 🏗 **software-engineering** | [[software-engineering/software-engineering]] | ✅ **사전 완성** — SOLID/Clean Code/TDD/DDD |
| 🎨 **design-patterns** | [[design-patterns/design-patterns]] | ✅ **사전 완성** — GoF 23 + PoEAA + FP/Concurrent/MSA |
| 🌐 **distributed-systems** | [[distributed-systems/distributed-systems]] | ✅ **사전 완성** — CAP/Consensus/CRDT/Saga |

### Tier 3 — 고급 / 시니어 면접

| 영역 | 진입점 | 상태 |
| --- | --- | --- |
| 🏛 **computer-architecture** | [[computer-architecture/computer-architecture]] | ✅ **사전 완성** — Von Neumann/ISA/Pipeline/Cache |
| 🔐 **security-theory** | [[security-theory/security-theory]] | ✅ **사전 완성** — Crypto/OAuth/OWASP/Zero Trust |
| 🛠 **compiler** | [[compiler/compiler]] | ✅ **사전 완성** — Lexer/Parser/IR/LLVM/JIT/GC |
| ⚙️ **hardware** | [[hardware/hardware]] | ✅ **사전 완성** — Transistor/Memory/Storage/SoC |
| 🗄 database (기존) | [[database/]] | 레거시 보존 |

---

## 📊 사전 완성 통계

| 메트릭 | 값 |
| --- | --- |
| 영역 수 | **13** |
| 알고리즘 노트 | 8 (그리디 / 구현 / DFS-BFS / 정렬 / 이진 탐색 / DP / 최단 경로 / 그래프 이론) |
| 자료구조 노트 | 9 (배열 / 리스트 / 스택큐 / 트리 / 힙 / 해시 / 그래프 / 트라이 / 고급) |
| CS 이론 노트 | 8 (네트워크 / OS / DB / 분산 / SW공학 / 패턴 / 보안 / CompArch / 컴파일러 / HW) |
| 전체 줄 수 | **약 13,000+** |
| 코드 언어 | 11 (Python / Java / C++ / C / JS / TS / Kotlin / Swift / Go / Rust / C#) |
| 대표 문제 풀이 | 50+ |
| 학습 자료 인용 | 100+ 책 / 강의 / 시각화 |

---

## 학습 경로 — 8 단계 로드맵

### 1 단계 (1-3 개월) — 코딩 테스트 통과
1. [[algorithm/algorithm]] 의 PART 02 (8 챕터) 정독
2. [[data-structure/data-structure]] 9 영역 — 배열 / 리스트 / 스택큐 / 트리 / 힙 / 해시 / 그래프 / 트라이 / 고급
3. 백준 단계별 → 실버 ~ 골드 5

### 2 단계 (1-2 개월) — 시스템 기초
4. [[operating-system/operating-system]] — 프로세스 / 스레드 / 메모리 / FS / I/O
5. [[network/network]] — TCP/IP / HTTP / TLS / DNS
6. [[database-theory/database-theory]] — 인덱스 / 트랜잭션 / 정규화 / CAP

### 3 단계 (1-2 개월) — 소프트웨어 공학
7. [[software-engineering/software-engineering]] — SOLID / 테스트 / 리팩터링 / DDD
8. [[design-patterns/design-patterns]] — GoF + 아키텍처 + 함수형 + 분산

### 4 단계 (2-3 개월) — 분산 / 시니어 면접
9. [[distributed-systems/distributed-systems]] — CAP / Raft / DDIA 책
10. 시스템 설계 면접 — [[../../00-inbox/links/general/system-design|↗ system-design 링크]]

### 5 단계 (선택) — 깊이 추구
11. [[computer-architecture/computer-architecture]] — 파이프라인 / 캐시 / SIMD
12. [[security-theory/security-theory]] — 암호학 / OAuth / OWASP
13. [[compiler/compiler]] — Lexer / Parser / LLVM / GC
14. [[hardware/hardware]] — 트랜지스터 / DRAM / SoC

---

## 사전 수준 노트 표준 (14 섹션 + 11 언어)

모든 노트가 따르는 표준 (~800-1100 줄):

1. **한 줄 정의**
2. **역사 / 배경** (인물, 연도, 핵심 통찰)
3. **개념의 깊이** (이론 / ADT / 하드웨어 상호작용)
4. **핵심 동작 원리** (예시 데이터 / 그림 / 수식 증명)
5. **복잡도 분석**
6. **의사 코드**
7. **언어별 구현 11 개** — Python / Java / C++ / C / JavaScript / TypeScript / Kotlin / Swift / Go / Rust / C#
8. **변형 / 응용**
9. **함정 / 안티패턴**
10. **실전 풀이 절차**
11. **대표 문제 + 풀이 코드**
12. **연습 문제 (난이도별)**
13. **학습 자료** (책 / 강의 / 시각화)
14. **관련 hub** (cross-link)

각 영역별 추가 깊이:
- **algorithm** → 동작 원리 + 11 언어 구현 + 대표 문제
- **data-structure** → ADT 이론 + 메모리 레이아웃 + 학술 변형
- **OS** → 메커니즘 + 시스템 콜 + 도구 (strace)
- **network** → 프로토콜 + 패킷 + 도구 (dig/tcpdump/curl)
- **DB 이론** → SQL / 인덱스 / 트랜잭션 + EXPLAIN 실습

---

## 운영 규칙

1. **사전이 목표** — 한 노트로 그 개념을 처음부터 끝까지 이해 가능해야.
2. **실습 가능** — 모든 노트에 "직접 따라할 코드 / 명령어".
3. **11 언어 구현** — 알고리즘 / 자료구조 중심.
4. **단순 복붙 금지** — 내 해석 / 정리 / 적용 예 50% 이상.
5. **출처 + 날짜 필수**.
6. **CS 이론 ↔ 실무 cross-link** — `database-theory/transaction` ↔ `database/postgresql/transaction-tuning`.

---

## 영역별 핵심 키워드 표

빠른 검색 / 면접 대비:

| 영역 | 핵심 키워드 |
| --- | --- |
| algorithm | 그리디, 구현, DFS/BFS, 정렬, 이진 탐색, DP, Dijkstra, 위상정렬 |
| data-structure | 배열, 캐시 라인, ADT, B+Tree, MVCC, Hash, MESI, DSU |
| network | TCP/IP, HTTP/3, QUIC, TLS 1.3, DNS, BGP, CORS, CDN |
| operating-system | Process/Thread, Fork+CoW, MLFQ/CFS, MVCC, mmap, epoll, io_uring |
| database-theory | Codd, ACID, BCNF, MVCC, B+Tree, LSM-Tree, CAP, PACELC |
| software-engineering | SOLID, GRASP, TDD, DDD, Bounded Context, Clean Architecture |
| design-patterns | GoF 23, Singleton, Strategy, Observer, PoEAA, Saga, CQRS |
| distributed-systems | CAP, Paxos, Raft, CRDT, 2PC/Saga/TCC, Outbox, Lamport, TrueTime |
| computer-architecture | Von Neumann, RISC vs CISC, Pipeline, Branch Prediction, MESI, SIMD |
| security-theory | CIA, AES-GCM, bcrypt/Argon2, OAuth 2.0, JWT, OWASP Top 10, mTLS |
| compiler | Lexer, LL/LR, AST, SSA, LLVM, JIT, GC, Hindley-Milner |
| hardware | MOSFET, SRAM/DRAM, NAND Flash, PCIe, HBM, SoC, Apple M |

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

---

## data-structure 완성 보고

9 영역 — 사전 완성 (~5000 줄):

| 영역 | 노트 | 핵심 |
| --- | --- | --- |
| 배열 / 문자열 | [[data-structure/arrays-and-strings/arrays-and-strings]] | ~1100 줄, Liskov ADT, 캐시 라인, SIMD, Grapheme |
| 연결 리스트 | [[data-structure/linked-lists/linked-lists]] | LRU, Floyd Cycle, Skip List 기반 |
| 스택 / 큐 | [[data-structure/stacks-and-queues/stacks-and-queues]] | Monotonic, Min Stack, Deque |
| 트리 | [[data-structure/trees/trees]] | BST/AVL/RB/B-Tree/B+Tree/LSM-Tree |
| 힙 | [[data-structure/heaps/heaps]] | Build Heap O(N) Floyd 1964, Fibonacci/Pairing |
| 해시 테이블 | [[data-structure/hash-tables/hash-tables]] | Chaining/OA, Universal Hashing, Swiss Table |
| 그래프 | [[data-structure/graphs/graphs]] | Matrix/List/Edge List, DAG/Tree/Bipartite |
| 트라이 | [[data-structure/tries/tries]] | Prefix Tree, Radix, Aho-Corasick |
| 고급 | [[data-structure/advanced/advanced]] | DSU, Segment Tree, BIT, Bloom Filter, Skip List |

---

## CS 이론 8 영역 완성 보고

| 영역 | 노트 | 핵심 |
| --- | --- | --- |
| 네트워크 | [[network/network]] | OSI, TCP 3-way, HTTP/3, TLS 1.3, DNS, CORS |
| 운영체제 | [[operating-system/operating-system]] | CFS, MVCC, epoll/io_uring, Buddy/Slab, Container |
| DB 이론 | [[database-theory/database-theory]] | Codd 1970, Index 8 종, MVCC, CAP/PACELC |
| 분산 시스템 | [[distributed-systems/distributed-systems]] | FLP, Paxos/Raft, CRDT, 2PC/Saga, Outbox |
| 소프트웨어 공학 | [[software-engineering/software-engineering]] | SOLID, TDD/BDD, Hexagonal/DDD/Clean |
| 디자인 패턴 | [[design-patterns/design-patterns]] | GoF 23, PoEAA, Concurrent, MSA |
| 보안 이론 | [[security-theory/security-theory]] | CIA, AES-GCM, Argon2, OAuth 2.0, OWASP Top 10 |
| 컴퓨터 구조 | [[computer-architecture/computer-architecture]] | Pipeline, Branch Prediction, MESI, SIMD, NUMA |
| 컴파일러 | [[compiler/compiler]] | Lexer/Parser, SSA, LLVM, JIT, GC, Hindley-Milner |
| 하드웨어 | [[hardware/hardware]] | MOSFET, DRAM, NAND, PCIe, HBM, SoC |

---

## 관련

- [[../areas|↑ 20-areas]]
- [[../../00-inbox/links/general/system-design|↗ system-design 링크 카탈로그]]
- [[../../00-inbox/links/general/books|↗ CS 책 카탈로그]]
- [[../../00-inbox/links/engineering/backend/database/database|↗ database 실무 링크]]
