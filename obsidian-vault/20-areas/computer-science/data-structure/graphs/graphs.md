---
title: "그래프 (Graphs)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T08:30:00+09:00
tags:
  - data-structure
  - graph
---

# 그래프 (Graphs)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 수준 |

**[[../data-structure|↑ data-structure]]** · **[[../../computer-science|↑↑ computer-science]]**

> 이 노트는 **자료구조 측면** (표현 / 메모리 / 순회). 알고리즘 (DFS/BFS/Dijkstra/MST) 은
> [[../../algorithm/dfs-bfs/dfs-bfs]] 와 [[../../algorithm/graph-theory/graph-theory]] 에서.

---

## 1. 한 줄 정의

**노드 (Vertex) + 간선 (Edge)** 으로 이루어진 가장 일반적인 자료구조. 임의의
두 노드가 직접 연결 가능. 트리 / 리스트 / 격자는 모두 그래프의 특수 케이스.

---

## 2. 개념의 깊이

### 2.1 그래프는 어디에나

- **SNS 친구 관계** — 무방향 그래프
- **웹 페이지 링크** — 방향 그래프 (PageRank 의 토대)
- **지도 / 도로** — 가중치 그래프
- **소프트웨어 의존성** — DAG
- **신경망** — 노드 = 뉴런, 간선 = 연결
- **분자 구조** — 원자 + 결합
- **컴파일러의 IR / SSA** — DAG

### 2.2 용어

- **노드 (Vertex, V)** / **간선 (Edge, E)**
- **방향 그래프 (Directed)** — 간선에 방향. 트위터 팔로우, 의존성.
- **무방향 그래프 (Undirected)** — 방향 없음. 페이스북 친구.
- **가중치 (Weighted)** — 간선에 비용. 도로 거리.
- **차수 (Degree)** — 한 노드의 이웃 수.
- **경로 (Path)** — 노드들의 연속 시퀀스.
- **사이클 (Cycle)** — 시작 = 끝 인 경로.
- **DAG** (Directed Acyclic Graph) — 방향 + 사이클 없음. 위상 정렬 가능.
- **트리** = 사이클 없는 무방향 연결 그래프.
- **이분 그래프 (Bipartite)** — 노드를 두 그룹으로 나눠 같은 그룹 내 간선 없음.

### 2.3 그래프 표현 3 가지

#### (a) 인접 행렬 (Adjacency Matrix)
```
N × N 행렬. A[i][j] = 1 (간선 있음) / 0 (없음). 가중치면 비용 저장.

장점: i ↔ j 간선 여부 O(1).
단점: 공간 O(V²). 희소 그래프에서 메모리 폭발.
사용: V ≤ 500 정도의 dense 그래프.
```

#### (b) 인접 리스트 (Adjacency List)
```
각 노드마다 이웃의 리스트.
1: [2, 3, 5]
2: [1, 4]
...

장점: 공간 O(V + E). 희소 그래프 효율.
단점: i ↔ j 간선 확인 O(degree).
사용: 대부분의 경우.
```

#### (c) 간선 리스트 (Edge List)
```
[(1,2,w), (2,3,w), ...]

장점: Kruskal MST 처럼 간선 정렬 필요할 때.
사용: 알고리즘에 따라.
```

### 2.4 비교

| 연산 | 인접 행렬 | 인접 리스트 |
| --- | --- | --- |
| 간선 추가 | O(1) | O(1) |
| 간선 삭제 | O(1) | O(degree) |
| 간선 존재 확인 | O(1) | O(degree) |
| 한 노드 이웃 순회 | O(V) | O(degree) |
| 공간 | O(V²) | O(V + E) |
| dense | ✅ | ⚠️ |
| sparse | ⚠️ 메모리 낭비 | ✅ |

### 2.5 메모리 추정

V = 10^5, E = 10^5 (sparse):
- 인접 행렬: 10^10 bit = 1.25 GB → 불가
- 인접 리스트: O(2 × 10^5) = 800KB → OK

V = 500, E = V² / 4 = 62500 (dense):
- 인접 행렬: 250 KB → OK
- 인접 리스트: 100 KB

---

## 3. 의사 코드 + 언어 (간략)

### Python — 인접 리스트
```python
from collections import defaultdict
graph = defaultdict(list)

# 무방향 + 가중치
def add_edge(u, v, w):
    graph[u].append((v, w))
    graph[v].append((u, w))   # 방향 그래프는 한 줄만

add_edge(1, 2, 5)
add_edge(2, 3, 3)
add_edge(1, 3, 4)

# 순회 (모든 이웃)
for v, w in graph[1]:
    print(f"1 → {v} (w={w})")
```

### Java
```java
Map<Integer, List<int[]>> graph = new HashMap<>();
for (int i = 0; i < V; i++) graph.put(i, new ArrayList<>());
graph.get(1).add(new int[]{2, 5});   // (v, w)
```

### C++ — 인접 리스트
```cpp
vector<vector<pair<int,int>>> graph(V);  // (v, w)
graph[1].push_back({2, 5});

// 인접 행렬
vector<vector<int>> adj(V, vector<int>(V, 0));
adj[1][2] = 5;
```

### Rust — 인접 리스트
```rust
let mut graph: Vec<Vec<(usize, i32)>> = vec![vec![]; n];
graph[1].push((2, 5));
```

(나머지 언어는 패턴 동일 — `Map<i, List<(v, w)>>`)

---

## 4. 주요 알고리즘 (간략 인덱스)

자세한 사전은 [[../../algorithm/algorithm|algorithm]] 참조.

| 알고리즘 | 시간 | 영역 |
| --- | --- | --- |
| DFS | O(V+E) | [[../../algorithm/dfs-bfs/dfs-bfs]] |
| BFS | O(V+E) | [[../../algorithm/dfs-bfs/dfs-bfs]] |
| Dijkstra | O((V+E) log V) | [[../../algorithm/shortest-path/shortest-path]] |
| Bellman-Ford | O(VE) | [[../../algorithm/shortest-path/shortest-path]] |
| Floyd-Warshall | O(V³) | [[../../algorithm/shortest-path/shortest-path]] |
| Kruskal MST | O(E log E) | [[../../algorithm/graph-theory/graph-theory]] |
| Prim MST | O((V+E) log V) | [[../../algorithm/graph-theory/graph-theory]] |
| Topological Sort | O(V+E) | [[../../algorithm/graph-theory/graph-theory]] |
| Tarjan SCC | O(V+E) | 강 연결 요소 |

---

## 5. 특수 그래프 / 변형

### (a) DAG (Directed Acyclic Graph)
사이클 없는 방향 그래프. 위상 정렬, 의존성, 빌드 시스템.

### (b) Tree
연결된 사이클 없는 그래프. 별도 [[../trees/trees]].

### (c) Bipartite Graph
2 그룹 분할. 매칭, 컬러링.

### (d) Complete Graph
모든 쌍 연결. K_N.

### (e) Planar Graph
평면에 교차 없이 그릴 수 있는 그래프. 4색 정리.

### (f) Multigraph
같은 두 노드 간 여러 간선.

### (g) Hypergraph
간선이 2 개 초과 노드 연결.

---

## 6. 함정 / 안티패턴

### 함정 1 — 무방향 / 방향 혼동
무방향이면 `add_edge(u, v)` + `add_edge(v, u)` 둘 다. 방향이면 한 줄만.

### 함정 2 — 인접 행렬 메모리 폭발
V ≥ 10^4 면 행렬 X. 항상 리스트.

### 함정 3 — visited 표시 시점
BFS 큐에 넣자마자 visited[v]=true. 나중에 표시하면 중복 큐 입력.

### 함정 4 — 자기 루프 / 평행 간선
그래프 정의에 따라 다름. 문제 조건 확인.

### 함정 5 — 거대 그래프의 DFS 재귀
스택 오버플로. 명시적 스택 또는 `sys.setrecursionlimit`.

### 함정 6 — Edge weight 가 0 이면
`if w` 같은 조건으로 0 weight edge 무시 가능. 명시적 비교.

---

## 7. 실전 절차

1. **표현 결정** — V, E 크기로 행렬 vs 리스트.
2. **방향 / 무방향**.
3. **가중치**.
4. **시작점 / 끝점** — 단일 / 모든 쌍.
5. **알고리즘 선택** — DFS/BFS/Dijkstra/Floyd/Kruskal.

---

## 8. 대표 문제

### Q1. Number of Islands (LeetCode 200) — DFS
```python
def num_islands(grid):
    if not grid: return 0
    n, m = len(grid), len(grid[0])
    count = 0
    def dfs(i, j):
        if i<0 or i>=n or j<0 or j>=m or grid[i][j] != '1': return
        grid[i][j] = '0'
        for di, dj in [(-1,0),(1,0),(0,-1),(0,1)]: dfs(i+di, j+dj)
    for i in range(n):
        for j in range(m):
            if grid[i][j] == '1':
                count += 1; dfs(i, j)
    return count
```

### Q2. Course Schedule (LeetCode 207) — 위상 정렬 (사이클 감지)
```python
from collections import defaultdict, deque
def can_finish(n, prereqs):
    graph = defaultdict(list)
    indeg = [0] * n
    for a, b in prereqs:
        graph[b].append(a); indeg[a] += 1
    q = deque(i for i in range(n) if indeg[i] == 0)
    visited = 0
    while q:
        u = q.popleft(); visited += 1
        for v in graph[u]:
            indeg[v] -= 1
            if indeg[v] == 0: q.append(v)
    return visited == n
```

### Q3. Clone Graph (LeetCode 133) — DFS + Hash
```python
def clone_graph(node):
    if not node: return None
    visited = {}
    def dfs(node):
        if node in visited: return visited[node]
        clone = Node(node.val)
        visited[node] = clone
        for n in node.neighbors:
            clone.neighbors.append(dfs(n))
        return clone
    return dfs(node)
```

### Q4. Network Delay Time (LeetCode 743) — Dijkstra
[[../../algorithm/shortest-path/shortest-path]] 참조.

---

## 9. 연습 문제

### 입문
- [LeetCode 200 Number of Islands](https://leetcode.com/problems/number-of-islands/)
- [LeetCode 133 Clone Graph](https://leetcode.com/problems/clone-graph/)
- [LeetCode 463 Island Perimeter](https://leetcode.com/problems/island-perimeter/)

### 중급
- [LeetCode 207 Course Schedule](https://leetcode.com/problems/course-schedule/)
- [LeetCode 210 Course Schedule II](https://leetcode.com/problems/course-schedule-ii/)
- [LeetCode 994 Rotting Oranges](https://leetcode.com/problems/rotting-oranges/)

### 고급
- [LeetCode 743 Network Delay Time](https://leetcode.com/problems/network-delay-time/)
- [LeetCode 787 Cheapest Flights](https://leetcode.com/problems/cheapest-flights-within-k-stops/)
- [백준 1197 최소 스패닝 트리](https://www.acmicpc.net/problem/1197)

---

## 10. 학습 자료

- **CLRS Part VI** — Graph Algorithms
- **Graph Algorithms (Mark Needham)** — Neo4j 의 실용서
- [VisuAlgo — Graph DS](https://visualgo.net/en/graphds)
- [Algorithms 4th — Graphs](https://algs4.cs.princeton.edu/40graphs/)

---

## 11. 관련

- [[../trees/trees]] — 사이클 없는 무방향 그래프
- [[../../algorithm/dfs-bfs/dfs-bfs]]
- [[../../algorithm/shortest-path/shortest-path]]
- [[../../algorithm/graph-theory/graph-theory]]
- [[../data-structure|↑ data-structure]]
