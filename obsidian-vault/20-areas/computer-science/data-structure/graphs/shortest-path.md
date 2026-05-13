---
title: "Shortest Path — Dijkstra / Bellman-Ford / Floyd"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:15:00+09:00
tags:
  - data-structure
  - graph
  - shortest-path
---

# Shortest Path

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Dijkstra / Bellman / Floyd / A* |

**[[graphs|↑ Graphs]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

가중치 그래프에서 — 두 노드 사이 **최소 가중치 path**.

---

## 2. 알고리즘 — 한눈

| 알고리즘 | 시간 | 음수 가중치 | 음수 cycle | 용도 |
| --- | --- | --- | --- | --- |
| **BFS** | O(V+E) | X (unweighted) | - | Unweighted |
| **Dijkstra** | O((V+E) log V) | X | - | Non-negative |
| **Bellman-Ford** | O(VE) | OK | 감지 | 음수 가능 |
| **SPFA** | O(VE) worst, 평균 빠름 | OK | 감지 | Bellman-Ford 빠른 변형 |
| **Floyd-Warshall** | O(V³) | OK | 감지 | All pairs |
| **Johnson** | O(V² log V + VE) | OK | 감지 | All pairs sparse |
| **A*** | depends on h | X | - | Heuristic |

---

## 3. Dijkstra

```python
import heapq

def dijkstra(graph, start):
    dist = {n: float('inf') for n in graph}
    dist[start] = 0
    pq = [(0, start)]
    while pq:
        d, u = heapq.heappop(pq)
        if d > dist[u]: continue
        for v, w in graph[u]:
            new_d = d + w
            if new_d < dist[v]:
                dist[v] = new_d
                heapq.heappush(pq, (new_d, v))
    return dist
```

### 시간 — O((V+E) log V) (binary heap)
- Fibonacci heap — O(E + V log V)
- 실제 — binary heap 빠름

### 조건
- 모든 edge 가중치 ≥ 0

### Path 복원
```python
parent = {n: None for n in graph}
# Dijkstra 안에서:
if new_d < dist[v]:
    parent[v] = u
    ...
```

---

## 4. Bellman-Ford

```python
def bellman_ford(graph, start):
    dist = {n: float('inf') for n in graph}
    dist[start] = 0
    
    # V-1 번 relax
    for _ in range(len(graph) - 1):
        for u in graph:
            for v, w in graph[u]:
                if dist[u] + w < dist[v]:
                    dist[v] = dist[u] + w
    
    # V 번째 relax 가능 — 음수 cycle
    for u in graph:
        for v, w in graph[u]:
            if dist[u] + w < dist[v]:
                return None    # 음수 cycle
    
    return dist
```

### 시간 — O(V × E)

### 효과
- 음수 가중치 OK
- 음수 cycle 감지

### 사용
- 환율 (음수 weight)
- Routing protocols (RIP, BGP 일부)

---

## 5. SPFA — Shortest Path Faster Algorithm

### 정의
- Bellman-Ford 의 queue-based 변형
- 변경된 노드만 큐에 넣음

```python
from collections import deque

def spfa(graph, start):
    dist = {n: float('inf') for n in graph}
    dist[start] = 0
    q = deque([start])
    in_queue = {start}
    
    while q:
        u = q.popleft()
        in_queue.remove(u)
        for v, w in graph[u]:
            if dist[u] + w < dist[v]:
                dist[v] = dist[u] + w
                if v not in in_queue:
                    q.append(v)
                    in_queue.add(v)
    return dist
```

### 효과
- 평균 — 매우 빠름
- 최악 — O(VE) (Bellman-Ford 와 같음)
- 음수 cycle 검사 — count

---

## 6. Floyd-Warshall — All Pairs

```python
def floyd_warshall(graph_matrix):
    n = len(graph_matrix)
    dist = [row[:] for row in graph_matrix]
    
    for k in range(n):
        for i in range(n):
            for j in range(n):
                if dist[i][k] + dist[k][j] < dist[i][j]:
                    dist[i][j] = dist[i][k] + dist[k][j]
    
    return dist
```

### 시간 — O(V³)
### 공간 — O(V²)

### 효과
- 모든 쌍 — 한 번에
- 음수 가중치 OK
- 음수 cycle — dist[i][i] < 0 인 i

### 사용
- 작은 dense graph
- Transitive closure (0/1)

---

## 7. Johnson — Sparse all-pairs

### 정의
- Bellman-Ford 한 번 (re-weight)
- Dijkstra V 번 (각 source)

### 시간 — O(V² log V + VE)

### 효과
- Sparse + 음수 — Floyd 보다 빠름

---

## 8. A* — Heuristic Search

### 정의
- Priority — f(n) = g(n) + h(n)
  - g — 시작에서 n 까지 actual cost
  - h — n 에서 goal 까지 estimated cost (heuristic)

```python
def a_star(graph, start, goal, h):
    pq = [(h(start), 0, start)]    # (f, g, node)
    visited = set()
    while pq:
        f, g, node = heapq.heappop(pq)
        if node == goal: return g
        if node in visited: continue
        visited.add(node)
        for nei, w in graph[node]:
            if nei not in visited:
                heapq.heappush(pq, (g + w + h(nei), g + w, nei))
    return -1
```

### 조건
- h(n) — admissible (실제 cost 이하)
- consistent — h(u) ≤ w(u,v) + h(v)

### 사용
- 게임 pathfinding
- Navigation (Google Maps + 추가 heuristic)
- AI search

---

## 9. Bidirectional Dijkstra

### 정의
- Source + target — 양쪽에서 동시
- 만나면 path 후보

### 효과
- 큰 graph — 시간 ↓

### 사용
- OSM (OpenStreetMap)
- Road navigation

---

## 10. Contraction Hierarchies

### 정의
- 노드 — importance 순서로 contract
- Shortcut edges 만들기

### 효과
- 백만 노드 — μs 단위 query
- 사용 — OSM, OSRM

자세히 → algorithm-advanced

---

## 11. Negative-weight cycle

### 정의
- Cycle 의 weight 총합 < 0
- 무한 loop — 더 짧은 path

### 감지
- Bellman-Ford — V 번째 relax 가능?
- Floyd — dist[i][i] < 0?
- SPFA — 한 노드 V 번 이상 큐 진입?

---

## 12. 0-1 BFS

### 정의
- Edge weight ∈ {0, 1}
- Deque — 0 edge: push_front, 1: push_back

### 시간 — O(V + E)
- Dijkstra 와 같은 결과, 빠름

자세히 → [[bfs]]

---

## 13. 응용

### Navigation (Google Maps / Waze)
- 도로 — graph, 거리 / 시간 — weight
- Bidirectional + Contraction Hierarchies

### Routing protocol
- OSPF — Dijkstra (link state)
- RIP — Bellman-Ford (distance vector)
- BGP — path vector

자세히 → [[../../network/routing/bgp]]

### Currency exchange
- 환율 → 음수 log = weight
- Negative cycle = arbitrage 가능

### Game AI
- A* — pathfinding

### Network flow
- 최소 cost flow — successive shortest path

---

## 14. 함정

### 함정 1 — Dijkstra 의 음수 가중치
잘못된 결과. Bellman-Ford 필요.

### 함정 2 — Floyd 의 메모리
V = 10000 — 100 MB.

### 함정 3 — Bellman-Ford 의 음수 cycle reach
음수 cycle 이 source 에서 도달 가능해야 — dist[i] = -inf.

### 함정 4 — Dijkstra 의 stale entry
Lazy delete — `if d > dist[u]: continue`.

### 함정 5 — Path 복원의 메모리
모든 노드 — parent. 큰 graph 메모리 ↑.

### 함정 6 — A* 의 heuristic 잘못
Inadmissible (overestimate) — 잘못된 path 반환.

---

## 15. 학습 자료

- CLRS Ch 24-25
- "Algorithms" Sedgewick
- "Network Flows" (Ahuja)
- Google Maps blog (shortest path)

---

## 16. 관련

- [[graphs]] — Hub
- [[bfs]] / [[dfs]]
- [[../heaps/heap-applications]] — Priority queue
- [[../../network/routing/bgp]]
