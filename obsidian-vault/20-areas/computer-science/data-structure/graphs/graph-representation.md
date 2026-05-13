---
title: "Graph Representation — 표현 방식"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:00:00+09:00
tags:
  - data-structure
  - graph
---

# Graph Representation

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Matrix vs List / 가중치 |

**[[graphs|↑ Graphs]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

Graph (V, E) — **노드 + 간선**. 표현 방법 — Adjacency List / Matrix / Edge List.

---

## 2. 3 가지 표현

### 2.1 Adjacency List
```python
graph = {
    1: [2, 3],
    2: [1, 4],
    3: [1],
    4: [2]
}
# 또는 list of lists
graph = [[2, 3], [1, 4], [1], [2]]
```

### 2.2 Adjacency Matrix
```python
graph = [
    [0, 1, 1, 0],
    [1, 0, 0, 1],
    [1, 0, 0, 0],
    [0, 1, 0, 0]
]
# graph[i][j] == 1 — edge i→j
```

### 2.3 Edge List
```python
edges = [(1, 2), (1, 3), (2, 4)]
```

---

## 3. 시공간 비교

| | Adjacency List | Adjacency Matrix | Edge List |
| --- | --- | --- | --- |
| **공간** | O(V+E) | O(V²) | O(E) |
| **Edge check** | O(degree) | O(1) | O(E) |
| **Neighbors** | O(degree) | O(V) | O(E) |
| **Sparse graph** | Good | Wasteful | OK |
| **Dense graph** | OK | Good | Bad |

### Sparse vs Dense
- **Sparse** — E ≈ V — list
- **Dense** — E ≈ V² — matrix

---

## 4. Directed vs Undirected

### Directed
```
1 → 2
adjacency: {1: [2]}
```

### Undirected
```
1 — 2
adjacency: {1: [2], 2: [1]}    # 양쪽
```

---

## 5. Weighted

### Adjacency list
```python
graph = {
    1: [(2, 5), (3, 10)],    # (neighbor, weight)
    2: [(1, 5), (4, 7)],
    ...
}
```

### Matrix
```python
graph = [
    [0, 5, 10, INF],
    [5, 0, INF, 7],
    ...
]
# INF — 연결 없음
```

---

## 6. 특수 — Multigraph / Self-loop

### Multigraph
- 두 노드 사이 — 여러 edge
- 다중 transit (도로 + 철도)

### Self-loop
- 노드 → 자기

### Adjacency list — 자연
- `1: [2, 2, 3]` (1→2 두 번)

### Matrix — 카운트로
- `matrix[1][2] = 2`

---

## 7. CSR / CSC — Compact Sparse Row/Column

### 정의
- 대용량 sparse graph
- 메모리 적음 + cache 친화

```
graph:
  1: [2, 3]
  2: [4]
  3: [4]
  4: []

CSR:
  ptr =     [0, 2, 3, 4, 4]
  indices = [2, 3, 4, 4]
```

### 사용
- BFS / DFS / SpMV
- 큰 graph (Twitter / Facebook)
- GraphBLAS

---

## 8. Adjacency Map (Hash)

### 사용
- 노드 ID 가 dense X (random string)
- 동적 graph

```python
graph = defaultdict(list)
graph["alice"].append("bob")
graph["alice"].append("charlie")
```

---

## 9. Implicit Graph

### 정의
- 명시적으로 저장 X
- 함수로 — `neighbors(node)` 동적 계산

### 예
- Grid (보드 게임) — 노드 = 좌표, edge = 인접 4 칸
- Rubik's Cube — state space
- Sliding puzzle

### BFS / DFS — 같은 방식 동작

---

## 10. Property — Graph Theory

| 용어 | 의미 |
| --- | --- |
| **Degree** | 노드의 인접 수 |
| **Path** | 노드 sequence (edge 연결) |
| **Cycle** | 시작 = 끝 |
| **Connected** | 모든 노드 연결 |
| **DAG** | Directed Acyclic Graph |
| **Tree** | Connected + no cycle |
| **Forest** | Acyclic (multiple trees) |
| **Bipartite** | 2 색 가능 (2-coloring) |
| **Complete** | 모든 쌍 — edge |

---

## 11. 응용

### Social Network
- Facebook / Twitter
- Friend / Follow graph

### 도로 / 교통
- 도로 → edge, 교차로 → node
- Dijkstra / A*

### 의존성 (Build / Make)
- 파일 / 모듈 — DAG
- Topological sort

### Web
- HTML 페이지 — node
- Link — edge

### State machine
- State — node
- Transition — edge

### 분자 / 회로
- Atom / 부품 — node
- Bond / wire — edge

---

## 12. 메모리 — 큰 graph

### Twitter 전체 — 약 1B users
- Adjacency list — 약 100 GB+

### 도구
- GraphX (Spark)
- Neo4j (graph DB)
- TigerGraph
- Apache Giraph (BSP)

---

## 13. 함정

### 함정 1 — Undirected 양쪽 추가 잊음
- `graph[a].append(b)` 만 — 단방향 됨.

### 함정 2 — 자기 자신 visit
방문 set 으로 — 무한 loop 방지.

### 함정 3 — 노드 ID 의 통일성
정수 vs 문자열. 매핑 dict.

### 함정 4 — Multigraph 의 weight
같은 두 노드 — 여러 weight. tuple list.

### 함정 5 — Sparse 가정의 dense graph
ML / 추천 — dense. Matrix.

### 함정 6 — 큰 노드 ID
ID gap — dict 가 list 보다 효율.

---

## 14. 학습 자료

- CLRS Ch 22
- "Algorithms" Sedgewick — Graph
- "Networks" — Mark Newman

---

## 15. 관련

- [[graphs]] — Hub
- [[bfs]] / [[dfs]] — 순회
- [[shortest-path]] — Dijkstra
