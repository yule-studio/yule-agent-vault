---
title: "Topological Sort — 위상 정렬"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:25:00+09:00
tags:
  - data-structure
  - graph
  - dag
---

# Topological Sort

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Kahn / DFS / 응용 |

**[[graphs|↑ Graphs]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

**DAG (Directed Acyclic Graph)** 의 노드를 — 모든 edge u→v 가 u 가 v 앞에 오도록 정렬.

---

## 2. 조건 — DAG

### Cycle 있으면 — 불가
- A → B → C → A
- 어느 순서도 위반

### DAG 만 — Topological order 존재

---

## 3. Kahn's Algorithm (BFS)

```python
from collections import deque

def topo_sort(graph):
    in_degree = {n: 0 for n in graph}
    for n in graph:
        for nei in graph[n]:
            in_degree[nei] += 1
    
    q = deque([n for n in graph if in_degree[n] == 0])
    result = []
    
    while q:
        node = q.popleft()
        result.append(node)
        for nei in graph[node]:
            in_degree[nei] -= 1
            if in_degree[nei] == 0:
                q.append(nei)
    
    if len(result) != len(graph):
        return None    # cycle
    return result
```

### 시간 — O(V + E)
### Cycle 감지 — Yes

---

## 4. DFS-based

```python
def topo_sort_dfs(graph):
    visited = set()
    on_stack = set()
    result = []
    
    def dfs(node):
        if node in on_stack: return False    # cycle
        if node in visited: return True
        on_stack.add(node)
        for nei in graph[node]:
            if not dfs(nei): return False
        on_stack.remove(node)
        visited.add(node)
        result.append(node)
        return True
    
    for n in graph:
        if n not in visited:
            if not dfs(n): return None    # cycle
    
    return result[::-1]    # reverse post-order
```

### 시간 — O(V + E)

---

## 5. Kahn vs DFS

| | Kahn (BFS) | DFS |
| --- | --- | --- |
| **자료구조** | Queue | 재귀 / Stack |
| **순서** | Layer 별 | Post-order 역순 |
| **Cycle** | result.size < V | on_stack 재방문 |
| **Memory** | in_degree array | recursion + on_stack |
| **Multi-source** | Easy | 자연 |
| **Stable** | Layer 안 임의 | DFS 순서 |

### 결과 — 유일 X
- 일반적으로 — 여러 topological order 가능

---

## 6. 응용

### 6.1 Build / Make
- 파일 의존성
- C/C++ — header, object files
- npm / pip / cargo — package dependencies

### 6.2 Course schedule
- 선수 과목 → 후속 과목

```python
def can_finish(num_courses, prerequisites):
    graph = defaultdict(list)
    in_degree = [0] * num_courses
    for course, pre in prerequisites:
        graph[pre].append(course)
        in_degree[course] += 1
    
    q = deque([i for i in range(num_courses) if in_degree[i] == 0])
    taken = 0
    while q:
        c = q.popleft()
        taken += 1
        for nei in graph[c]:
            in_degree[nei] -= 1
            if in_degree[nei] == 0:
                q.append(nei)
    return taken == num_courses
```

### 6.3 Spreadsheet — Formula evaluation
- Cell A1 = B1 + C1
- 의존성 — DAG
- Topological order — recalc

### 6.4 Task scheduling
- Job dependency
- Apache Airflow DAG

### 6.5 Compiler — Type checking
- Class 의 dependency
- Forward declaration 순서

### 6.6 Symbol resolution
- Linker — 의존성

---

## 7. Lexicographically smallest

### 문제
- 가능한 모든 topological order 중 — 사전식 가장 작은

### 해결 — Kahn + Min-heap
- Queue 대신 — heap
- 항상 가장 작은 in_degree=0 노드 먼저

```python
import heapq

def lex_smallest_topo(graph, n):
    in_degree = [0] * n
    for u in graph:
        for v in graph[u]:
            in_degree[v] += 1
    
    heap = [i for i in range(n) if in_degree[i] == 0]
    heapq.heapify(heap)
    
    result = []
    while heap:
        node = heapq.heappop(heap)
        result.append(node)
        for nei in graph[node]:
            in_degree[nei] -= 1
            if in_degree[nei] == 0:
                heapq.heappush(heap, nei)
    
    return result if len(result) == n else None
```

---

## 8. Parallel / Layer-based Scheduling

### Layer
- Same in_degree=0 — 동시에 수행 가능
- Wave 별 처리

```python
def layered_topo(graph, n):
    in_degree = [0] * n
    for u in graph:
        for v in graph[u]:
            in_degree[v] += 1
    
    layers = []
    current = [i for i in range(n) if in_degree[i] == 0]
    while current:
        layers.append(current)
        next_layer = []
        for node in current:
            for nei in graph[node]:
                in_degree[nei] -= 1
                if in_degree[nei] == 0:
                    next_layer.append(nei)
        current = next_layer
    return layers
```

### 사용
- Build parallelism (make -j)
- DAG scheduler (Spark, Airflow)

---

## 9. Critical Path Method (CPM)

### 정의
- Topological order + duration 합산
- 가장 긴 path — bottleneck

### 사용
- Project management
- Build optimization

---

## 10. Cycle Detection

### Topological sort 자체 — cycle 감지
- Kahn: result.size != V
- DFS: gray node 재방문

### 별도 — 3-color DFS
- WHITE / GRAY / BLACK

자세히 → [[dfs]]

---

## 11. SCC + Topological Sort

### 정의
- Directed graph (cycle 있음) — SCC 로 압축 → DAG → topological sort

### 사용
- 컴파일러 — recursion 그룹
- Web graph — popularity

---

## 12. 함정

### 함정 1 — 이름 / id 의 자료형
정수 vs 문자열. dict.

### 함정 2 — Self-loop
A → A — cycle. Kahn 결과 < V.

### 함정 3 — Multi-edge
같은 edge 여러 번 — in_degree 다중 ↑.

### 함정 4 — Disconnected
한 큐로 모두 처리됨 (각 component 의 source 가 큐에).

### 함정 5 — 결과 unique X
한 답 만 검증 — 다양한 답 OK.

### 함정 6 — Cycle 의 의미
"Course Schedule II" 등 — 모든 노드 처리 되어야 valid.

---

## 13. 학습 자료

- CLRS Ch 22.4
- LeetCode Topological Sort
- "Algorithms" Sedgewick — Topological

---

## 14. 관련

- [[graphs]] — Hub
- [[bfs]] / [[dfs]]
- [[../../algorithm/algorithm]] — DAG / Critical path
