---
title: "그래프 이론 (Graph Theory)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T03:30:00+09:00
tags:
  - algorithm
  - graph
  - mst
  - union-find
  - topological-sort
---

# 그래프 이론 (Graph Theory)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 수준 + 11 언어 (이코테 Ch.10/18) |

**[[../algorithm|↑ algorithm]]** · **[[../computer-science/computer-science|↑↑ computer-science]]**

---

## 1. 한 줄 정의

DFS/BFS / 최단 경로 외의 **그래프 알고리즘 종합** — 서로소 집합 (Union-Find) /
최소 신장 트리 (MST) / 위상 정렬 (Topological Sort) / 사이클 감지.

---

## 2. 그래프 이론 — 3 핵심 주제

### (a) 서로소 집합 (Disjoint Set / Union-Find)

**여러 원소를 그룹으로 묶고, 같은 그룹인지 빠르게 판정**.
- 사이클 감지 / Kruskal MST / 네트워크 연결 판정.
- `find(x)`: x 가 속한 그룹의 대표 반환.
- `union(a, b)`: a, b 그룹 병합.
- 시간: `α(N)` (역 아커만 함수) ≈ O(1).

### (b) 최소 신장 트리 (MST)

**모든 노드를 연결하는 최소 가중치 부분 그래프** (사이클 없음, V-1 간선).

| 알고리즘 | 방식 | 시간 |
| --- | --- | --- |
| **Kruskal** | 간선 가중치 작은 순 + Union-Find | `O(E log E)` |
| **Prim** | 시작점부터 가까운 간선 추가 (Dijkstra 유사) | `O((V+E) log V)` |

E ≤ V² 면 Kruskal 이 보통 더 간단. V 가 크고 E 가 sparse 면 Prim 도 OK.

### (c) 위상 정렬 (Topological Sort)

**DAG (방향 비순환 그래프)** 의 노드를 **선행 관계 순서**로 나열.
- "선수 과목 → 후속 과목" 학습 순서.
- "프로젝트 의존성" 빌드 순서.

| 방식 | 자료구조 | 시간 |
| --- | --- | --- |
| **Kahn's (BFS)** | 진입 차수 + queue | `O(V+E)` |
| **DFS 역순** | 재귀 + 후위 탐색 | `O(V+E)` |

---

## 3. 핵심 직관

### Union-Find — "리더가 누구냐?"
- 각 노드는 `parent` 를 가짐. 같은 그룹은 같은 root.
- `find(x)` = root 찾아 올라감 (경로 압축).
- `union(a, b)` = a 의 root 를 b 의 root 의 자식으로 (Union by Rank).

### Kruskal MST — "싼 간선부터 추가, 사이클 만들면 거부"
- 간선 정렬.
- Union-Find 로 사이클 검사.
- V-1 간선 추가하면 끝.

### Prim MST — "현재 트리에서 가장 가까운 노드 흡수"
- 시작 노드 1 개.
- 매번 트리 ↔ 미방문 노드 간 최소 가중치 간선 선택.
- min-heap 활용 → Dijkstra 와 거의 같음.

### 위상 정렬 — "진입 차수 0 인 노드부터"
- 진입 차수 (in-degree) = 0 인 노드 → 큐.
- 큐에서 꺼내며 그 노드의 후속 진입 차수 -1.
- 후속이 0 되면 큐에.

---

## 4. 동작 원리 — 예시

### Union-Find 예시

```
초기: parent = [0, 1, 2, 3, 4, 5]   # 모두 자기 자신이 root

union(1, 2):
  find(1) = 1, find(2) = 2
  parent[1] = 2 → parent = [0, 2, 2, 3, 4, 5]

union(3, 4):
  parent = [0, 2, 2, 4, 4, 5]

union(2, 4):
  find(2) = 2, find(4) = 4
  parent[2] = 4 → parent = [0, 2, 4, 4, 4, 5]
  (이제 1, 2, 3, 4 모두 4 가 root)

same_group(1, 3) → find(1) = 4, find(3) = 4 → True
```

**경로 압축**: `find` 하면서 거쳐간 모든 노드의 parent 를 root 로 직접 연결.

### Kruskal MST 예시

```
간선 (V=4, E=5):
  (1,2,1), (2,3,2), (3,4,2), (1,4,5), (1,3,3)

정렬 (가중치 순):
  (1,2,1), (2,3,2), (3,4,2), (1,3,3), (1,4,5)

처리:
  (1,2,1): find(1)≠find(2) → union. MST: {(1,2)}, cost=1
  (2,3,2): find(2)≠find(3) → union. MST: {(1,2),(2,3)}, cost=3
  (3,4,2): find(3)≠find(4) → union. MST: {(1,2),(2,3),(3,4)}, cost=5
  (1,3,3): find(1)=find(3) → 사이클! 거부.
  (1,4,5): find(1)=find(4) → 사이클! 거부.

V-1 = 3 간선 채워서 종료. cost=5
```

### 위상 정렬 예시 (Kahn's)

```
DAG:
  1 → 2 → 4
  1 → 3 → 4
  3 → 5
  4 → 5

진입 차수:
  1: 0
  2: 1 (from 1)
  3: 1 (from 1)
  4: 2 (from 2, 3)
  5: 2 (from 3, 4)

Step 1: 진입차수 0 인 {1} → 큐.
Step 2: pop 1. result=[1]. 2, 3 진입차수 -1 → {2:0, 3:0}. 큐 = [2, 3].
Step 3: pop 2. result=[1,2]. 4 진입 차수 -1 → 1. 큐 = [3].
Step 4: pop 3. result=[1,2,3]. 4, 5 -1 → 4:0, 5:1. 큐 = [4].
Step 5: pop 4. result=[1,2,3,4]. 5 -1 → 0. 큐 = [5].
Step 6: pop 5. result=[1,2,3,4,5]. 종료.

위상 정렬: 1 → 2 → 3 → 4 → 5
```

---

## 5. 복잡도

| 알고리즘 | 시간 | 공간 |
| --- | --- | --- |
| Union-Find (경로 압축 + Union by Rank) | `α(N)` ≈ O(1) per op | `O(N)` |
| Kruskal MST | `O(E log E)` | `O(V+E)` |
| Prim MST (heap) | `O((V+E) log V)` | `O(V+E)` |
| 위상 정렬 (Kahn's) | `O(V+E)` | `O(V+E)` |
| 사이클 감지 (DFS) | `O(V+E)` | `O(V)` |

---

## 6. 의사 코드

### Union-Find
```
parent[i] ← i for all i

function find(x):
    if parent[x] == x: return x
    parent[x] ← find(parent[x])   # 경로 압축
    return parent[x]

function union(a, b):
    ra ← find(a); rb ← find(b)
    if ra == rb: return
    parent[rb] ← ra   # 또는 rank 작은 쪽을 큰 쪽 아래로
```

### Kruskal MST
```
function KRUSKAL(V, edges):
    sort edges by weight
    init Union-Find of V nodes
    mst ← []
    for (u, v, w) in edges:
        if find(u) ≠ find(v):
            union(u, v)
            mst.append((u, v, w))
            if len(mst) == V - 1: break
    return mst
```

### Topological Sort (Kahn's)
```
function TOPO(V, edges):
    indegree[*] ← 0
    for (u, v) in edges: indegree[v] += 1
    queue ← all nodes with indegree 0
    result ← []
    while queue not empty:
        u ← pop
        result.append(u)
        for v in graph[u]:
            indegree[v] -= 1
            if indegree[v] == 0: queue.push(v)
    if len(result) < V: return "cycle exists"
    return result
```

---

## 7. 언어별 구현 — Union-Find + Kruskal + Topological Sort (11 언어)

### Python 3
```python
class UnionFind:
    def __init__(self, n):
        self.parent = list(range(n + 1))
        self.rank = [0] * (n + 1)
    def find(self, x):
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])
        return self.parent[x]
    def union(self, a, b):
        ra, rb = self.find(a), self.find(b)
        if ra == rb: return False
        if self.rank[ra] < self.rank[rb]: ra, rb = rb, ra
        self.parent[rb] = ra
        if self.rank[ra] == self.rank[rb]: self.rank[ra] += 1
        return True

def kruskal(n, edges):
    uf = UnionFind(n)
    edges.sort(key=lambda e: e[2])
    mst, cost = [], 0
    for u, v, w in edges:
        if uf.union(u, v):
            mst.append((u, v, w)); cost += w
    return mst, cost

from collections import deque
def topo_sort(n, edges):
    graph = [[] for _ in range(n + 1)]
    indeg = [0] * (n + 1)
    for u, v in edges:
        graph[u].append(v); indeg[v] += 1
    q = deque(i for i in range(1, n + 1) if indeg[i] == 0)
    result = []
    while q:
        u = q.popleft(); result.append(u)
        for v in graph[u]:
            indeg[v] -= 1
            if indeg[v] == 0: q.append(v)
    return result if len(result) == n else None
```

### Java
```java
import java.util.*;

class UnionFind {
    int[] parent, rank;
    UnionFind(int n) {
        parent = new int[n + 1]; rank = new int[n + 1];
        for (int i = 0; i <= n; i++) parent[i] = i;
    }
    int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]);
        return parent[x];
    }
    boolean union(int a, int b) {
        int ra = find(a), rb = find(b);
        if (ra == rb) return false;
        if (rank[ra] < rank[rb]) { int t = ra; ra = rb; rb = t; }
        parent[rb] = ra;
        if (rank[ra] == rank[rb]) rank[ra]++;
        return true;
    }
}

public class GraphTheory {
    static int kruskal(int n, int[][] edges) {
        Arrays.sort(edges, (a, b) -> a[2] - b[2]);
        UnionFind uf = new UnionFind(n);
        int cost = 0;
        for (int[] e : edges) {
            if (uf.union(e[0], e[1])) cost += e[2];
        }
        return cost;
    }

    static List<Integer> topoSort(int n, int[][] edges) {
        List<List<Integer>> graph = new ArrayList<>();
        int[] indeg = new int[n + 1];
        for (int i = 0; i <= n; i++) graph.add(new ArrayList<>());
        for (int[] e : edges) {
            graph.get(e[0]).add(e[1]); indeg[e[1]]++;
        }
        Queue<Integer> q = new ArrayDeque<>();
        for (int i = 1; i <= n; i++) if (indeg[i] == 0) q.offer(i);
        List<Integer> result = new ArrayList<>();
        while (!q.isEmpty()) {
            int u = q.poll(); result.add(u);
            for (int v : graph.get(u)) {
                if (--indeg[v] == 0) q.offer(v);
            }
        }
        return result.size() == n ? result : null;
    }
}
```

### C++
```cpp
#include <bits/stdc++.h>
using namespace std;

struct UF {
    vector<int> parent, rank_;
    UF(int n) : parent(n + 1), rank_(n + 1, 0) {
        iota(parent.begin(), parent.end(), 0);
    }
    int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]);
        return parent[x];
    }
    bool unite(int a, int b) {
        int ra = find(a), rb = find(b);
        if (ra == rb) return false;
        if (rank_[ra] < rank_[rb]) swap(ra, rb);
        parent[rb] = ra;
        if (rank_[ra] == rank_[rb]) rank_[ra]++;
        return true;
    }
};

int kruskal(int n, vector<vector<int>>& edges) {
    sort(edges.begin(), edges.end(), [](auto& a, auto& b){ return a[2] < b[2]; });
    UF uf(n);
    int cost = 0;
    for (auto& e : edges) {
        if (uf.unite(e[0], e[1])) cost += e[2];
    }
    return cost;
}

vector<int> topo_sort(int n, vector<pair<int,int>>& edges) {
    vector<vector<int>> graph(n + 1);
    vector<int> indeg(n + 1, 0);
    for (auto& [u, v] : edges) { graph[u].push_back(v); indeg[v]++; }
    queue<int> q;
    for (int i = 1; i <= n; i++) if (indeg[i] == 0) q.push(i);
    vector<int> result;
    while (!q.empty()) {
        int u = q.front(); q.pop();
        result.push_back(u);
        for (int v : graph[u]) if (--indeg[v] == 0) q.push(v);
    }
    return (int)result.size() == n ? result : vector<int>{};
}
```

### C
```c
#include <stdio.h>
#include <stdlib.h>

int parent[100005], rnk[100005];

void init(int n) {
    for (int i = 0; i <= n; i++) { parent[i] = i; rnk[i] = 0; }
}
int find(int x) {
    if (parent[x] != x) parent[x] = find(parent[x]);
    return parent[x];
}
int unite(int a, int b) {
    int ra = find(a), rb = find(b);
    if (ra == rb) return 0;
    if (rnk[ra] < rnk[rb]) { int t = ra; ra = rb; rb = t; }
    parent[rb] = ra;
    if (rnk[ra] == rnk[rb]) rnk[ra]++;
    return 1;
}

typedef struct { int u, v, w; } Edge;
int cmp(const void *a, const void *b) { return ((Edge*)a)->w - ((Edge*)b)->w; }

int kruskal(int n, Edge edges[], int m) {
    qsort(edges, m, sizeof(Edge), cmp);
    init(n);
    int cost = 0;
    for (int i = 0; i < m; i++) {
        if (unite(edges[i].u, edges[i].v)) cost += edges[i].w;
    }
    return cost;
}
```

### JavaScript (Node.js)
```javascript
class UnionFind {
    constructor(n) {
        this.parent = Array.from({length: n + 1}, (_, i) => i);
        this.rank = new Array(n + 1).fill(0);
    }
    find(x) {
        if (this.parent[x] !== x) this.parent[x] = this.find(this.parent[x]);
        return this.parent[x];
    }
    union(a, b) {
        let ra = this.find(a), rb = this.find(b);
        if (ra === rb) return false;
        if (this.rank[ra] < this.rank[rb]) [ra, rb] = [rb, ra];
        this.parent[rb] = ra;
        if (this.rank[ra] === this.rank[rb]) this.rank[ra]++;
        return true;
    }
}

function kruskal(n, edges) {
    edges.sort((a, b) => a[2] - b[2]);
    const uf = new UnionFind(n);
    let cost = 0;
    for (const [u, v, w] of edges) if (uf.union(u, v)) cost += w;
    return cost;
}

function topoSort(n, edges) {
    const graph = Array.from({length: n + 1}, () => []);
    const indeg = new Array(n + 1).fill(0);
    for (const [u, v] of edges) { graph[u].push(v); indeg[v]++; }
    const queue = [];
    for (let i = 1; i <= n; i++) if (indeg[i] === 0) queue.push(i);
    const result = [];
    while (queue.length) {
        const u = queue.shift(); result.push(u);
        for (const v of graph[u]) if (--indeg[v] === 0) queue.push(v);
    }
    return result.length === n ? result : null;
}
```

### TypeScript
```typescript
class UnionFind {
    parent: number[]; rank: number[];
    constructor(n: number) {
        this.parent = Array.from({length: n + 1}, (_, i) => i);
        this.rank = new Array(n + 1).fill(0);
    }
    find(x: number): number {
        if (this.parent[x] !== x) this.parent[x] = this.find(this.parent[x]);
        return this.parent[x];
    }
    union(a: number, b: number): boolean {
        let ra = this.find(a), rb = this.find(b);
        if (ra === rb) return false;
        if (this.rank[ra] < this.rank[rb]) [ra, rb] = [rb, ra];
        this.parent[rb] = ra;
        if (this.rank[ra] === this.rank[rb]) this.rank[ra]++;
        return true;
    }
}
```

### Kotlin
```kotlin
class UnionFind(n: Int) {
    val parent = IntArray(n + 1) { it }
    val rank = IntArray(n + 1)
    fun find(x: Int): Int {
        if (parent[x] != x) parent[x] = find(parent[x])
        return parent[x]
    }
    fun union(a: Int, b: Int): Boolean {
        var ra = find(a); var rb = find(b)
        if (ra == rb) return false
        if (rank[ra] < rank[rb]) { val t = ra; ra = rb; rb = t }
        parent[rb] = ra
        if (rank[ra] == rank[rb]) rank[ra]++
        return true
    }
}

fun kruskal(n: Int, edges: Array<IntArray>): Int {
    edges.sortBy { it[2] }
    val uf = UnionFind(n)
    var cost = 0
    for (e in edges) if (uf.union(e[0], e[1])) cost += e[2]
    return cost
}
```

### Swift
```swift
class UnionFind {
    var parent: [Int]; var rank: [Int]
    init(_ n: Int) { parent = Array(0...n); rank = Array(repeating: 0, count: n + 1) }
    func find(_ x: Int) -> Int {
        if parent[x] != x { parent[x] = find(parent[x]) }
        return parent[x]
    }
    func union(_ a: Int, _ b: Int) -> Bool {
        var ra = find(a), rb = find(b)
        if ra == rb { return false }
        if rank[ra] < rank[rb] { (ra, rb) = (rb, ra) }
        parent[rb] = ra
        if rank[ra] == rank[rb] { rank[ra] += 1 }
        return true
    }
}

func kruskal(_ n: Int, _ edges: [[Int]]) -> Int {
    let sorted = edges.sorted { $0[2] < $1[2] }
    let uf = UnionFind(n)
    var cost = 0
    for e in sorted { if uf.union(e[0], e[1]) { cost += e[2] } }
    return cost
}
```

### Go
```go
package main

import (
    "fmt"
    "sort"
)

type UF struct {
    parent, rank []int
}

func NewUF(n int) *UF {
    uf := &UF{parent: make([]int, n+1), rank: make([]int, n+1)}
    for i := range uf.parent { uf.parent[i] = i }
    return uf
}
func (uf *UF) Find(x int) int {
    if uf.parent[x] != x { uf.parent[x] = uf.Find(uf.parent[x]) }
    return uf.parent[x]
}
func (uf *UF) Union(a, b int) bool {
    ra, rb := uf.Find(a), uf.Find(b)
    if ra == rb { return false }
    if uf.rank[ra] < uf.rank[rb] { ra, rb = rb, ra }
    uf.parent[rb] = ra
    if uf.rank[ra] == uf.rank[rb] { uf.rank[ra]++ }
    return true
}

func Kruskal(n int, edges [][3]int) int {
    sort.Slice(edges, func(i, j int) bool { return edges[i][2] < edges[j][2] })
    uf := NewUF(n)
    cost := 0
    for _, e := range edges {
        if uf.Union(e[0], e[1]) { cost += e[2] }
    }
    return cost
}

func main() {
    edges := [][3]int{{1,2,1},{2,3,2},{3,4,2},{1,4,5},{1,3,3}}
    fmt.Println(Kruskal(4, edges))   // 5
}
```

### Rust
```rust
struct UF {
    parent: Vec<usize>,
    rank: Vec<usize>,
}

impl UF {
    fn new(n: usize) -> Self {
        Self { parent: (0..=n).collect(), rank: vec![0; n + 1] }
    }
    fn find(&mut self, x: usize) -> usize {
        if self.parent[x] != x {
            self.parent[x] = self.find(self.parent[x]);
        }
        self.parent[x]
    }
    fn union(&mut self, a: usize, b: usize) -> bool {
        let (mut ra, mut rb) = (self.find(a), self.find(b));
        if ra == rb { return false; }
        if self.rank[ra] < self.rank[rb] { std::mem::swap(&mut ra, &mut rb); }
        self.parent[rb] = ra;
        if self.rank[ra] == self.rank[rb] { self.rank[ra] += 1; }
        true
    }
}

fn kruskal(n: usize, mut edges: Vec<(usize, usize, i32)>) -> i32 {
    edges.sort_by_key(|e| e.2);
    let mut uf = UF::new(n);
    let mut cost = 0;
    for (u, v, w) in edges {
        if uf.union(u, v) { cost += w; }
    }
    cost
}
```

### C#
```csharp
using System;
using System.Linq;
using System.Collections.Generic;

public class UnionFind {
    int[] parent, rank;
    public UnionFind(int n) {
        parent = new int[n + 1]; rank = new int[n + 1];
        for (int i = 0; i <= n; i++) parent[i] = i;
    }
    public int Find(int x) {
        if (parent[x] != x) parent[x] = Find(parent[x]);
        return parent[x];
    }
    public bool Union(int a, int b) {
        int ra = Find(a), rb = Find(b);
        if (ra == rb) return false;
        if (rank[ra] < rank[rb]) { int t = ra; ra = rb; rb = t; }
        parent[rb] = ra;
        if (rank[ra] == rank[rb]) rank[ra]++;
        return true;
    }
}

public class GraphTheory {
    public static int Kruskal(int n, int[][] edges) {
        Array.Sort(edges, (a, b) => a[2] - b[2]);
        var uf = new UnionFind(n);
        int cost = 0;
        foreach (var e in edges) if (uf.Union(e[0], e[1])) cost += e[2];
        return cost;
    }
}
```

---

## 8. Prim MST (Python 예시)

```python
import heapq
def prim(graph, start, n):
    visited = [False] * (n + 1)
    pq = [(0, start)]
    cost = 0
    while pq:
        w, u = heapq.heappop(pq)
        if visited[u]: continue
        visited[u] = True
        cost += w
        for v, weight in graph[u]:
            if not visited[v]:
                heapq.heappush(pq, (weight, v))
    return cost
```

---

## 9. 변형 / 응용

### (a) 사이클 감지 (Union-Find)
간선 추가할 때 두 정점이 이미 같은 그룹이면 사이클.

### (b) 사이클 감지 (DFS)
방향 그래프: `visited` 3 색 (미방문/처리중/완료). 처리중 노드 또 발견 시 사이클.

### (c) 강 연결 요소 (SCC)
방향 그래프에서 서로 도달 가능한 노드 그룹. Tarjan / Kosaraju 알고리즘.

### (d) 이중 연결 요소
무방향 그래프에서 articulation point / bridge.

### (e) Lazy Propagation / Euler Tour
세그먼트 트리 응용 (고급).

### (f) 위상 정렬 + DP
DAG 의 최장 경로 / 최단 경로.

### (g) Spanning Tree 갯수 (Matrix Tree Theorem)
라플라시안 행렬의 cofactor.

---

## 10. 함정 / 안티패턴

### 함정 1 — `find` 없이 `parent[x]` 직접 비교
경로 압축 안 되면 트리 길어져 O(N) 가능. 항상 `find` 사용.

### 함정 2 — Union 시 rank 누락
rank 없으면 worst case 트리가 chain. O(log N) 보장 못 함.

### 함정 3 — Kruskal 에서 정렬 안 한 간선
"가중치 작은 순" 이 핵심. 정렬 빠뜨리면 결과 오답.

### 함정 4 — Topological Sort 의 사이클 감지
큐가 비었는데 결과 길이 < V → 사이클 존재. 반드시 체크.

### 함정 5 — 무방향 그래프에 위상 정렬
위상 정렬은 **DAG 전용**. 방향성 확인.

### 함정 6 — MST 가 unique 가 아님
가중치가 같으면 여러 MST 존재. 답 유일성 가정하지 말 것.

### 함정 7 — Prim 의 시작점 무관
어디서 시작해도 MST 비용 같음. 단지 트리 모양만 다름.

### 함정 8 — Kruskal 에서 V-1 간선 채우면 종료
나머지 무시 — 시간 절약.

---

## 11. 실전 풀이 절차 — 6 단계

1. **그래프 유형** — 무방향 / 방향 / 가중치 / DAG?
2. **목표 식별** — 연결 판정 / MST / 정렬 / 사이클 / SCC?
3. **자료구조** — UF / 인접 리스트 / heap.
4. **알고리즘 선택** — UF / Kruskal / Prim / Kahn's / Tarjan.
5. **간선 수 vs 노드 수** — Kruskal vs Prim 결정.
6. **사이클 / 다중 그룹 검증** — 결과 길이 확인.

---

## 12. 대표 문제 (이코테 Ch.10 + Ch.18)

### Q1. 팀 결성 (Ch.10) — Union-Find 기본

```python
n, m = map(int, input().split())
parent = list(range(n + 1))

def find(x):
    if parent[x] != x: parent[x] = find(parent[x])
    return parent[x]

def union(a, b):
    ra, rb = find(a), find(b)
    if ra > rb: ra, rb = rb, ra
    parent[rb] = ra

for _ in range(m):
    op, a, b = map(int, input().split())
    if op == 0: union(a, b)
    else: print('YES' if find(a) == find(b) else 'NO')
```

### Q2. 도시 분할 계획 (Ch.10) — MST + 마지막 간선 제거

```python
n, m = map(int, input().split())
edges = []
for _ in range(m):
    a, b, c = map(int, input().split())
    edges.append((c, a, b))
edges.sort()

parent = list(range(n + 1))
def find(x):
    if parent[x] != x: parent[x] = find(parent[x])
    return parent[x]

total = 0
last = 0
for w, a, b in edges:
    ra, rb = find(a), find(b)
    if ra != rb:
        parent[ra] = rb
        total += w
        last = w   # 가장 비싼 간선 기록

print(total - last)   # 마지막 간선 제거 = 2 마을
```

### Q3. 커리큘럼 (Ch.10) — 위상 정렬 + DP

```python
from collections import deque
n = int(input())
graph = [[] for _ in range(n + 1)]
indeg = [0] * (n + 1)
time = [0] * (n + 1)

for i in range(1, n + 1):
    data = list(map(int, input().split()))
    time[i] = data[0]
    for x in data[1:-1]:
        graph[x].append(i)
        indeg[i] += 1

dp = time[:]   # 누적 시간 = 본인 시간으로 초기화
q = deque(i for i in range(1, n + 1) if indeg[i] == 0)
while q:
    u = q.popleft()
    for v in graph[u]:
        dp[v] = max(dp[v], dp[u] + time[v])
        indeg[v] -= 1
        if indeg[v] == 0: q.append(v)

print('\n'.join(str(dp[i]) for i in range(1, n + 1)))
```

### Q4. 여행 계획 (Ch.18) — UF

```python
n, m = map(int, input().split())
parent = list(range(n + 1))
def find(x):
    if parent[x] != x: parent[x] = find(parent[x])
    return parent[x]

for i in range(1, n + 1):
    row = list(map(int, input().split()))
    for j in range(n):
        if row[j] == 1:
            a, b = find(i), find(j + 1)
            if a != b: parent[max(a, b)] = min(a, b)

plan = list(map(int, input().split()))
root = find(plan[0])
print('YES' if all(find(x) == root for x in plan) else 'NO')
```

### Q5. 행성 터널 (Ch.18) — MST 최적화
3D 좌표에서 두 행성 간 비용 = `min(|x1-x2|, |y1-y2|, |z1-z2|)`.

```python
n = int(input())
points = []
for i in range(n):
    x, y, z = map(int, input().split())
    points.append((x, y, z, i))

# x, y, z 각각 정렬해서 인접한 두 점만 간선으로 → 3(N-1) 간선
edges = []
for axis in range(3):
    sorted_pts = sorted(points, key=lambda p: p[axis])
    for i in range(n - 1):
        a, b = sorted_pts[i], sorted_pts[i + 1]
        edges.append((abs(a[axis] - b[axis]), a[3], b[3]))

edges.sort()
parent = list(range(n))
def find(x):
    if parent[x] != x: parent[x] = find(parent[x])
    return parent[x]

cost = 0
for w, a, b in edges:
    ra, rb = find(a), find(b)
    if ra != rb:
        parent[ra] = rb
        cost += w
print(cost)
```

---

## 13. 연습 문제 (난이도별)

### 입문 — Union-Find
- [백준 1717 집합의 표현](https://www.acmicpc.net/problem/1717)
- [백준 1976 여행 가자](https://www.acmicpc.net/problem/1976)

### 중급 — MST
- [백준 1197 최소 스패닝 트리](https://www.acmicpc.net/problem/1197)
- [백준 1922 네트워크 연결](https://www.acmicpc.net/problem/1922)
- [백준 4386 별자리 만들기](https://www.acmicpc.net/problem/4386)

### 중급 — 위상 정렬
- [백준 2252 줄 세우기](https://www.acmicpc.net/problem/2252)
- [백준 1766 문제집](https://www.acmicpc.net/problem/1766)

### 고급
- [백준 3665 최종 순위](https://www.acmicpc.net/problem/3665) — 위상 정렬 + 변경
- [백준 2150 SCC](https://www.acmicpc.net/problem/2150) — Tarjan
- [백준 4195 친구 네트워크](https://www.acmicpc.net/problem/4195) — UF + 해시

---

## 14. 학습 자료

- 이코테 **Ch.10, 18**
- [동빈나 — 그래프 이론](https://www.youtube.com/watch?v=aOhhNFTIeFI)
- [CLRS Ch.21-23](https://mitpress.mit.edu/9780262046305/) — Disjoint Sets / MST
- [VisuAlgo — Union-Find](https://visualgo.net/en/ufds)
- [VisuAlgo — MST](https://visualgo.net/en/mst)
- [Tarjan SCC 시각화](https://www.youtube.com/watch?v=wUgWX0nc4NY)

---

## 15. 관련

- [[../shortest-path/shortest-path]] — Dijkstra / Floyd-Warshall
- [[../dfs-bfs/dfs-bfs]] — 그래프 탐색 기초
- [[../greedy/greedy]] — Kruskal 의 그리디 본질
- [[../algorithm|↑ algorithm 인덱스]]
