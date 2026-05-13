---
title: "DFS — Depth-First Search"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:10:00+09:00
tags:
  - data-structure
  - graph
  - dfs
---

# DFS — Depth-First Search

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 재귀 / 반복 / 응용 |

**[[graphs|↑ Graphs]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

**한 path 끝까지 깊이 우선** — Stack (또는 재귀). Backtracking / Topological sort / SCC.

---

## 2. 알고리즘

### Recursive
```python
def dfs(graph, node, visited):
    if node in visited: return
    visited.add(node)
    process(node)
    for nei in graph[node]:
        dfs(graph, nei, visited)
```

### Iterative — explicit stack
```python
def dfs_iter(graph, start):
    visited = set()
    stack = [start]
    while stack:
        node = stack.pop()
        if node in visited: continue
        visited.add(node)
        process(node)
        for nei in graph[node]:
            if nei not in visited:
                stack.append(nei)
```

### 시간 — O(V + E)
### 공간 — O(V) (visited + recursion / stack)

---

## 3. Pre / Post-order

### Pre (enter time)
```python
def dfs_pre(graph, node, visited):
    visited.add(node)
    print(f"enter {node}")
    for nei in graph[node]:
        if nei not in visited:
            dfs_pre(graph, nei, visited)
```

### Post (exit time)
```python
def dfs_post(graph, node, visited):
    visited.add(node)
    for nei in graph[node]:
        if nei not in visited:
            dfs_post(graph, nei, visited)
    print(f"exit {node}")
```

### 사용
- Pre — building tree
- Post — Topological sort, SCC

---

## 4. Cycle Detection

### Undirected
```python
def has_cycle(graph):
    visited = set()
    def dfs(node, parent):
        visited.add(node)
        for nei in graph[node]:
            if nei not in visited:
                if dfs(nei, node): return True
            elif nei != parent:
                return True
        return False
    
    for n in graph:
        if n not in visited:
            if dfs(n, None): return True
    return False
```

### Directed — 3 colors
```
WHITE — 아직 안 봄
GRAY  — 처리 중 (재귀 stack 위)
BLACK — 완료
```

```python
def has_cycle_directed(graph):
    color = {n: 'WHITE' for n in graph}
    
    def dfs(node):
        color[node] = 'GRAY'
        for nei in graph[node]:
            if color[nei] == 'GRAY':
                return True    # back edge — cycle
            if color[nei] == 'WHITE' and dfs(nei):
                return True
        color[node] = 'BLACK'
        return False
    
    for n in graph:
        if color[n] == 'WHITE':
            if dfs(n): return True
    return False
```

---

## 5. Topological Sort — DFS

```python
def topo_sort(graph):
    visited = set()
    result = []
    
    def dfs(node):
        visited.add(node)
        for nei in graph[node]:
            if nei not in visited:
                dfs(nei)
        result.append(node)    # post-order
    
    for n in graph:
        if n not in visited:
            dfs(n)
    
    return result[::-1]    # reverse
```

자세히 → [[topological-sort]]

---

## 6. Strongly Connected Components (SCC)

### Tarjan's Algorithm
- DFS 한 번
- low-link value 계산
- Stack 으로 component 추출

### Kosaraju's Algorithm
- DFS 1: 원본 graph — finish order
- Graph 의 transpose
- DFS 2: 역순으로 — 각 DFS 가 한 SCC

```python
def kosaraju(graph):
    visited = set()
    order = []
    
    def dfs1(node):
        visited.add(node)
        for nei in graph[node]:
            if nei not in visited:
                dfs1(nei)
        order.append(node)
    
    for n in graph:
        if n not in visited:
            dfs1(n)
    
    # transpose
    transposed = defaultdict(list)
    for u in graph:
        for v in graph[u]:
            transposed[v].append(u)
    
    visited.clear()
    sccs = []
    
    def dfs2(node, component):
        visited.add(node)
        component.append(node)
        for nei in transposed[node]:
            if nei not in visited:
                dfs2(nei, component)
    
    for n in reversed(order):
        if n not in visited:
            comp = []
            dfs2(n, comp)
            sccs.append(comp)
    
    return sccs
```

---

## 7. Backtracking — DFS

### 정의
- 가능한 모든 시도 → 실패 시 — 이전 상태로

### N-Queens
```python
def solve_nqueens(n):
    result = []
    board = [-1] * n
    
    def is_safe(row, col):
        for i in range(row):
            if board[i] == col: return False
            if abs(board[i] - col) == row - i: return False
        return True
    
    def dfs(row):
        if row == n:
            result.append(board.copy())
            return
        for col in range(n):
            if is_safe(row, col):
                board[row] = col
                dfs(row + 1)
                board[row] = -1
    
    dfs(0)
    return result
```

### 다른 예
- Sudoku
- Permutations / Combinations
- Subsets
- Word Search (grid)

---

## 8. Connected Components

```python
def connected_components(graph):
    visited = set()
    components = []
    
    def dfs(node, component):
        visited.add(node)
        component.append(node)
        for nei in graph[node]:
            if nei not in visited:
                dfs(nei, component)
    
    for n in graph:
        if n not in visited:
            comp = []
            dfs(n, comp)
            components.append(comp)
    
    return components
```

### 사용
- Social network — 친구 그룹
- 이미지 — 같은 색 영역 (flood fill)

---

## 9. Flood Fill

```python
def flood_fill(image, sr, sc, new_color):
    old = image[sr][sc]
    if old == new_color: return image
    rows, cols = len(image), len(image[0])
    
    def dfs(r, c):
        if r < 0 or r >= rows or c < 0 or c >= cols: return
        if image[r][c] != old: return
        image[r][c] = new_color
        dfs(r-1, c); dfs(r+1, c); dfs(r, c-1); dfs(r, c+1)
    
    dfs(sr, sc)
    return image
```

### 사용
- Paint bucket (그림판)
- Number of Islands (LeetCode 200)

---

## 10. Bridge / Articulation Point

### Bridge
- 제거 시 — graph 분리되는 edge

### Tarjan algorithm — DFS
- low[node] = min(disc[node], disc[ancestor reachable via back edge])
- low[v] > disc[u] — (u, v) 가 bridge

---

## 11. Bipartite check (DFS)

```python
def is_bipartite(graph):
    color = {}
    for n in graph:
        if n not in color:
            stack = [(n, 0)]
            while stack:
                node, c = stack.pop()
                if node in color:
                    if color[node] != c: return False
                else:
                    color[node] = c
                    for nei in graph[node]:
                        stack.append((nei, 1 - c))
    return True
```

---

## 12. DFS vs BFS

| | DFS | BFS |
| --- | --- | --- |
| **구조** | Stack / 재귀 | Queue |
| **메모리** | Path 길이 만큼 | Level width 만큼 |
| **Shortest path** | X (Dijkstra) | O (unweighted) |
| **사이클** | 자연 | 가능 (parent 등) |
| **Topological** | Post-order | Kahn |
| **SCC** | Tarjan / Kosaraju | X |
| **Backtracking** | 자연 | X |

---

## 13. 함정

### 함정 1 — Recursion depth
큰 graph — stack overflow. Iterative.

### 함정 2 — Cycle 의 무한 loop
visited 체크.

### 함정 3 — Path reconstruction
parent[] / chain — backtrack.

### 함정 4 — 모든 노드 visit (multi-component)
한 시작 노드 만 — 한 component. 외부 loop.

### 함정 5 — Recursive 의 mutation
재귀 — 객체 변경 후 복원 (backtracking).

### 함정 6 — Disconnected graph
시작 노드 외에 — 별도 component. 모든 노드 외부 loop.

---

## 14. 학습 자료

- CLRS Ch 22
- LeetCode DFS / Backtracking tag
- "Algorithms" Sedgewick — Graph search

---

## 15. 관련

- [[graphs]] — Hub
- [[bfs]] — 비교
- [[topological-sort]] — Post-order DFS
- [[../../algorithm/algorithm]] — Backtracking
