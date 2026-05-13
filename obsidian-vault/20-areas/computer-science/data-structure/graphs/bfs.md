---
title: "BFS — Breadth-First Search"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:05:00+09:00
tags:
  - data-structure
  - graph
  - bfs
---

# BFS — Breadth-First Search

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 알고리즘 / 응용 / 변형 |

**[[graphs|↑ Graphs]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

**층별로 탐색** — 시작 노드의 가까운 노드부터 차례로. Queue 사용. Shortest path (unweighted).

---

## 2. 알고리즘

```python
from collections import deque

def bfs(graph, start):
    visited = {start}
    q = deque([start])
    while q:
        node = q.popleft()
        process(node)
        for nei in graph[node]:
            if nei not in visited:
                visited.add(nei)
                q.append(nei)
```

### 시간 — O(V + E)
### 공간 — O(V)

---

## 3. 시각화

```
     1
    / \
   2   3
  /|   |\
 4 5   6 7

BFS from 1:
  level 0: 1
  level 1: 2, 3
  level 2: 4, 5, 6, 7
```

### Queue 의 동작
```
queue: [1]                 visit 1
queue: [2, 3]              visit 2
queue: [3, 4, 5]           visit 3
queue: [4, 5, 6, 7]        visit 4
queue: [5, 6, 7]           visit 5
queue: [6, 7]              visit 6
queue: [7]                 visit 7
queue: []                  done
```

---

## 4. Shortest Path (Unweighted)

```python
def shortest_path(graph, start, end):
    visited = {start}
    q = deque([(start, 0)])    # (node, distance)
    while q:
        node, d = q.popleft()
        if node == end: return d
        for nei in graph[node]:
            if nei not in visited:
                visited.add(nei)
                q.append((nei, d + 1))
    return -1
```

### 보장
- BFS — 모든 edge 비용 동일 가정
- 첫 visit 시 — minimum distance

### Weighted — Dijkstra 사용

---

## 5. Path 복원

```python
def bfs_path(graph, start, end):
    parent = {start: None}
    q = deque([start])
    while q:
        node = q.popleft()
        if node == end:
            # reconstruct
            path = []
            while node:
                path.append(node)
                node = parent[node]
            return path[::-1]
        for nei in graph[node]:
            if nei not in parent:
                parent[nei] = node
                q.append(nei)
    return None
```

---

## 6. Multi-source BFS

### 정의
- 여러 source 동시 출발
- 각 노드 — 가장 가까운 source 의 거리

```python
def multi_source_bfs(grid):
    rows, cols = len(grid), len(grid[0])
    q = deque()
    dist = [[-1] * cols for _ in range(rows)]
    
    # 모든 source 시작
    for i in range(rows):
        for j in range(cols):
            if grid[i][j] == 'S':
                q.append((i, j))
                dist[i][j] = 0
    
    while q:
        x, y = q.popleft()
        for dx, dy in [(-1, 0), (1, 0), (0, -1), (0, 1)]:
            nx, ny = x + dx, y + dy
            if 0 <= nx < rows and 0 <= ny < cols and dist[nx][ny] == -1:
                dist[nx][ny] = dist[x][y] + 1
                q.append((nx, ny))
    return dist
```

### 사용
- "Rotting Oranges" (LeetCode 994)
- 여러 fire / virus 가 동시 퍼짐

---

## 7. 0-1 BFS

### 정의
- Edge weight 0 또는 1
- Deque 사용 — 0 edge: push_front, 1 edge: push_back

```python
def zero_one_bfs(graph, start):
    dist = {node: float('inf') for node in graph}
    dist[start] = 0
    dq = deque([start])
    while dq:
        node = dq.popleft()
        for nei, w in graph[node]:
            if dist[node] + w < dist[nei]:
                dist[nei] = dist[node] + w
                if w == 0:
                    dq.appendleft(nei)
                else:
                    dq.append(nei)
    return dist
```

### 시간 — O(V + E)
- Dijkstra 와 같은 결과 (Dijkstra: O((V+E) log V))

---

## 8. Bidirectional BFS

### 정의
- Start + End 양쪽에서 동시 BFS
- 만나면 path 완성

### 효과
- 시간 — O(b^(d/2)) (vs O(b^d) one-way)
- 큰 graph 에 효과 큼

### 예
- Word Ladder (LeetCode 127)

---

## 9. 응용

### 9.1 Maze / Grid
- 가장 짧은 path
- 4 / 8 방향

### 9.2 Knight's shortest path
- 체스 — 8 방향 knight move

### 9.3 Word Ladder
- 한 글자 변경 — 가장 짧은 transformation chain

### 9.4 Levels of tree
- Level-order traversal
- Bottom-up / Top-down

### 9.5 Bipartite check
- 2-coloring with BFS
- 인접 노드 — 다른 색

### 9.6 Connected components
- 각 component BFS

### 9.7 SNS — degrees of separation
- 6 degrees of Kevin Bacon
- BFS for friend distance

### 9.8 Web crawler
- BFS — 페이지 link 추적

### 9.9 Network broadcast
- Flood fill / 멀티캐스트

---

## 10. Topological sort — Kahn's algorithm

### BFS variant
```python
def topological_sort(graph):
    in_degree = {n: 0 for n in graph}
    for n in graph:
        for nei in graph[n]:
            in_degree[nei] += 1
    
    q = deque([n for n in graph if in_degree[n] == 0])
    result = []
    while q:
        n = q.popleft()
        result.append(n)
        for nei in graph[n]:
            in_degree[nei] -= 1
            if in_degree[nei] == 0:
                q.append(nei)
    
    if len(result) != len(graph):
        return None    # cycle 있음
    return result
```

자세히 → [[topological-sort]]

---

## 11. 함정

### 함정 1 — visited 빠뜨림
무한 loop. visit 즉시 mark.

### 함정 2 — Queue 의 효율
`list.pop(0)` O(N). deque 권장.

### 함정 3 — 큰 graph 의 memory
visited / parent — 큰 set. 압축 (bit array).

### 함정 4 — Multi-source 의 첫 mark
시작 시 모든 source — distance 0.

### 함정 5 — BFS for weighted
Edge weight 다르면 — Dijkstra. BFS 는 unweighted 만.

### 함정 6 — 시작 노드 visited 누락
visited.add(start) 잊으면 — 무한 loop (cycle 있는 graph).

---

## 12. 학습 자료

- CLRS Ch 22
- LeetCode BFS tag
- "Introduction to Algorithms" — Coursera

---

## 13. 관련

- [[graphs]] — Hub
- [[dfs]] — 비교
- [[shortest-path]] — Dijkstra (weighted)
- [[topological-sort]]
