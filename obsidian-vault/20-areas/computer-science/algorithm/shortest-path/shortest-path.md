---
title: "최단 경로 (Shortest Path)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T03:00:00+09:00
tags:
  - algorithm
  - shortest-path
  - graph
---

# 최단 경로 (Shortest Path)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 수준 + 11 언어 (이코테 Ch.09/17) |

**[[../algorithm|↑ algorithm]]** · **[[../computer-science/computer-science|↑↑ computer-science]]**

---

## 1. 한 줄 정의

**가중치가 있는 그래프에서 두 노드 간 최단 거리 / 경로**. 가중치가 없으면
BFS, 양수 가중치 → Dijkstra, 음수 포함 → Bellman-Ford, 모든 쌍 → Floyd-Warshall.

---

## 2. 최단 경로 알고리즘 — 4 가지 선택

| 알고리즘 | 가중치 | 시작점 수 | 음수 간선 | 시간 |
| --- | --- | --- | --- | --- |
| **BFS** | 모두 1 (없음) | 1 → All | ❌ | `O(V+E)` |
| **Dijkstra** | 양수 | 1 → All | ❌ | `O((V+E) log V)` with heap |
| **Bellman-Ford** | 음수 OK | 1 → All | ✅ + 음수 사이클 감지 | `O(V×E)` |
| **Floyd-Warshall** | 모두 | All → All | ✅ (음수 사이클 X) | `O(V³)` |

### 선택 기준

- **가중치 없음 + 1 시작** → BFS
- **양수 가중치 + 1 시작** → Dijkstra (가장 흔함)
- **음수 가중치 있음** → Bellman-Ford
- **모든 쌍 + V ≤ 500** → Floyd-Warshall (V³ ≈ 10^8)
- **DAG (사이클 없음)** → Topological Sort + DP

---

## 3. 핵심 직관

### Dijkstra — "가장 가까운 미방문 노드부터 확장"
- 모든 노드의 거리를 INF 로 초기화. 시작 = 0.
- 우선순위 큐에서 가장 가까운 미방문 노드 꺼냄.
- 그 노드의 이웃 거리 갱신 (relaxation).
- 반복 → 모든 노드 거리 확정.

**왜 그리디?** 한 번 확정된 거리는 다시 줄지 않는다 (양수 가중치). "이미
가장 짧은 거리로 확정된 노드를 통해 더 가까운 이웃 발견 가능".

### Floyd-Warshall — "중간 노드 K 를 거쳐 i → j 가 짧아지나?"
- 점화식: `D[i][j] = min(D[i][j], D[i][k] + D[k][j])`.
- 3중 반복으로 모든 쌍.

### Bellman-Ford — "모든 간선 V-1 번 relaxation"
- 음수 가중치 OK.
- V 번째에도 갱신 발생 → 음수 사이클 존재.

---

## 4. 동작 원리 — Dijkstra 예시

```
그래프 (V=5, E=8):
  1 → 2 (가중치 1)
  1 → 3 (가중치 2)
  1 → 4 (가중치 1)
  2 → 3 (가중치 5)
  2 → 4 (가중치 4)
  3 → 5 (가중치 1)
  4 → 3 (가중치 3)
  4 → 5 (가중치 7)

시작 = 1, 거리 = [∞, 0, ∞, ∞, ∞, ∞]
힙: [(0, 1)]

Step 1: pop (0, 1). 1 확정. 이웃 갱신:
  d[2] = 0+1 = 1
  d[3] = 0+2 = 2
  d[4] = 0+1 = 1
  힙: [(1, 2), (1, 4), (2, 3)]

Step 2: pop (1, 2). 2 확정. 이웃:
  d[3]: min(2, 1+5=6) = 2 (변경 X)
  d[4]: min(1, 1+4=5) = 1 (변경 X)
  힙: [(1, 4), (2, 3)]

Step 3: pop (1, 4). 4 확정. 이웃:
  d[3]: min(2, 1+3=4) = 2 (변경 X)
  d[5]: min(∞, 1+7=8) = 8
  힙: [(2, 3), (8, 5)]

Step 4: pop (2, 3). 3 확정. 이웃:
  d[5]: min(8, 2+1=3) = 3
  힙: [(3, 5), (8, 5)]

Step 5: pop (3, 5). 5 확정. (8, 5) 는 무시.

최종: d = [_, 0, 1, 2, 1, 3]
```

---

## 5. 복잡도

| 알고리즘 | 시간 | 공간 |
| --- | --- | --- |
| Dijkstra (인접 행렬 + 단순) | `O(V²)` | `O(V²)` |
| Dijkstra (인접 리스트 + heap) | `O((V+E) log V)` | `O(V+E)` |
| Bellman-Ford | `O(V×E)` | `O(V)` |
| Floyd-Warshall | `O(V³)` | `O(V²)` |
| SPFA (Bellman 최적화) | 평균 `O(E)`, 최악 `O(V×E)` | `O(V)` |
| 0-1 BFS | `O(V+E)` | `O(V)` |

---

## 6. 의사 코드

### Dijkstra (min-heap)
```
function DIJKSTRA(graph, start):
    dist[*] ← ∞
    dist[start] ← 0
    pq ← min-heap with (0, start)
    while pq not empty:
        d, u ← pop(pq)
        if d > dist[u]: continue   # 무시
        for (v, w) in graph[u]:
            if dist[u] + w < dist[v]:
                dist[v] ← dist[u] + w
                push(pq, (dist[v], v))
    return dist
```

### Floyd-Warshall
```
function FW(graph):
    initialize dist[i][j] = weight(i, j) (no edge → ∞, i=j → 0)
    for k in 1..V:
        for i in 1..V:
            for j in 1..V:
                dist[i][j] ← min(dist[i][j], dist[i][k] + dist[k][j])
    return dist
```

### Bellman-Ford
```
function BF(graph, start):
    dist[*] ← ∞; dist[start] ← 0
    for i in 1..V-1:
        for each edge (u, v, w):
            if dist[u] + w < dist[v]:
                dist[v] ← dist[u] + w
    # 음수 사이클 검사
    for each edge (u, v, w):
        if dist[u] + w < dist[v]: return "negative cycle"
    return dist
```

---

## 7. 언어별 구현 — Dijkstra (11 언어)

### Python 3
```python
import heapq
def dijkstra(graph, start, n):
    dist = [float('inf')] * (n + 1)
    dist[start] = 0
    pq = [(0, start)]
    while pq:
        d, u = heapq.heappop(pq)
        if d > dist[u]: continue
        for v, w in graph[u]:
            if dist[u] + w < dist[v]:
                dist[v] = dist[u] + w
                heapq.heappush(pq, (dist[v], v))
    return dist

graph = {
    1: [(2, 1), (3, 2), (4, 1)], 2: [(3, 5), (4, 4)],
    3: [(5, 1)], 4: [(3, 3), (5, 7)], 5: []
}
print(dijkstra(graph, 1, 5))   # [inf, 0, 1, 2, 1, 3]
```

### Java
```java
import java.util.*;
public class Dij {
    static int[] dijkstra(List<int[]>[] graph, int start, int n) {
        int[] dist = new int[n + 1];
        Arrays.fill(dist, Integer.MAX_VALUE);
        dist[start] = 0;
        PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);
        pq.offer(new int[]{0, start});
        while (!pq.isEmpty()) {
            int[] cur = pq.poll();
            int d = cur[0], u = cur[1];
            if (d > dist[u]) continue;
            for (int[] edge : graph[u]) {
                int v = edge[0], w = edge[1];
                if (dist[u] + w < dist[v]) {
                    dist[v] = dist[u] + w;
                    pq.offer(new int[]{dist[v], v});
                }
            }
        }
        return dist;
    }
}
```

### C++
```cpp
#include <bits/stdc++.h>
using namespace std;
const int INF = 1e9;

vector<int> dijkstra(vector<vector<pair<int,int>>>& graph, int start, int n) {
    vector<int> dist(n + 1, INF);
    dist[start] = 0;
    priority_queue<pair<int,int>, vector<pair<int,int>>, greater<>> pq;
    pq.push({0, start});
    while (!pq.empty()) {
        auto [d, u] = pq.top(); pq.pop();
        if (d > dist[u]) continue;
        for (auto& [v, w] : graph[u]) {
            if (dist[u] + w < dist[v]) {
                dist[v] = dist[u] + w;
                pq.push({dist[v], v});
            }
        }
    }
    return dist;
}
```

### C
```c
#include <stdio.h>
#include <limits.h>
#define V 6
// 인접 행렬 사용 (단순. heap 안 씀 — O(V²))

int min_dist(int dist[], int visited[]) {
    int min = INT_MAX, idx = -1;
    for (int i = 1; i < V; i++)
        if (!visited[i] && dist[i] < min) { min = dist[i]; idx = i; }
    return idx;
}

void dijkstra(int graph[V][V], int start) {
    int dist[V], visited[V] = {0};
    for (int i = 0; i < V; i++) dist[i] = INT_MAX;
    dist[start] = 0;
    for (int c = 0; c < V - 1; c++) {
        int u = min_dist(dist, visited);
        if (u == -1) break;
        visited[u] = 1;
        for (int v = 1; v < V; v++) {
            if (!visited[v] && graph[u][v] && dist[u] != INT_MAX
                && dist[u] + graph[u][v] < dist[v]) {
                dist[v] = dist[u] + graph[u][v];
            }
        }
    }
    for (int i = 1; i < V; i++) printf("%d ", dist[i]);
}
```

### JavaScript (Node.js)
```javascript
// Node.js 내장 min-heap 없음 → 간단 구현 또는 라이브러리
class MinHeap {
    constructor() { this.heap = []; }
    push(item) {
        this.heap.push(item);
        let i = this.heap.length - 1;
        while (i > 0) {
            const p = (i - 1) >> 1;
            if (this.heap[p][0] > this.heap[i][0]) {
                [this.heap[p], this.heap[i]] = [this.heap[i], this.heap[p]];
                i = p;
            } else break;
        }
    }
    pop() {
        const top = this.heap[0];
        const end = this.heap.pop();
        if (this.heap.length) {
            this.heap[0] = end;
            let i = 0;
            while (true) {
                const l = 2*i+1, r = 2*i+2;
                let s = i;
                if (l < this.heap.length && this.heap[l][0] < this.heap[s][0]) s = l;
                if (r < this.heap.length && this.heap[r][0] < this.heap[s][0]) s = r;
                if (s === i) break;
                [this.heap[i], this.heap[s]] = [this.heap[s], this.heap[i]];
                i = s;
            }
        }
        return top;
    }
    get size() { return this.heap.length; }
}

function dijkstra(graph, start, n) {
    const dist = new Array(n + 1).fill(Infinity);
    dist[start] = 0;
    const pq = new MinHeap(); pq.push([0, start]);
    while (pq.size > 0) {
        const [d, u] = pq.pop();
        if (d > dist[u]) continue;
        for (const [v, w] of graph[u] || []) {
            if (dist[u] + w < dist[v]) {
                dist[v] = dist[u] + w;
                pq.push([dist[v], v]);
            }
        }
    }
    return dist;
}
```

### TypeScript
```typescript
// 위 JS 구현과 동일. 타입만 부착.
type Edge = [number, number];
function dijkstra(graph: Map<number, Edge[]>, start: number, n: number): number[] {
    const dist: number[] = new Array(n + 1).fill(Infinity);
    dist[start] = 0;
    // (구현은 위 JS 와 같음)
    return dist;
}
```

### Kotlin
```kotlin
import java.util.PriorityQueue

fun dijkstra(graph: Array<MutableList<IntArray>>, start: Int, n: Int): IntArray {
    val dist = IntArray(n + 1) { Int.MAX_VALUE }
    dist[start] = 0
    val pq = PriorityQueue<IntArray>(compareBy { it[0] })
    pq.offer(intArrayOf(0, start))
    while (pq.isNotEmpty()) {
        val (d, u) = pq.poll()
        if (d > dist[u]) continue
        for (edge in graph[u]) {
            val (v, w) = edge
            if (dist[u] + w < dist[v]) {
                dist[v] = dist[u] + w
                pq.offer(intArrayOf(dist[v], v))
            }
        }
    }
    return dist
}
```

### Swift
```swift
// Swift 표준 라이브러리에 PriorityQueue 없음 → 직접 구현
struct Heap<T: Comparable> {
    private var items: [T] = []
    var isEmpty: Bool { items.isEmpty }
    mutating func push(_ x: T) {
        items.append(x); var i = items.count - 1
        while i > 0 {
            let p = (i - 1) / 2
            if items[p] > items[i] { items.swapAt(p, i); i = p } else { break }
        }
    }
    mutating func pop() -> T? {
        guard !items.isEmpty else { return nil }
        let top = items[0]; let end = items.removeLast()
        if !items.isEmpty { items[0] = end
            var i = 0
            while true {
                let l = 2*i+1, r = 2*i+2; var s = i
                if l < items.count && items[l] < items[s] { s = l }
                if r < items.count && items[r] < items[s] { s = r }
                if s == i { break }
                items.swapAt(i, s); i = s
            }
        }
        return top
    }
}

func dijkstra(_ graph: [Int: [(Int, Int)]], _ start: Int, _ n: Int) -> [Int] {
    var dist = Array(repeating: Int.max, count: n + 1)
    dist[start] = 0
    var pq = Heap<[Int]>()
    pq.push([0, start])
    while let cur = pq.pop() {
        let d = cur[0], u = cur[1]
        if d > dist[u] { continue }
        for (v, w) in graph[u] ?? [] {
            if dist[u] + w < dist[v] {
                dist[v] = dist[u] + w
                pq.push([dist[v], v])
            }
        }
    }
    return dist
}
```

### Go
```go
package main

import (
    "container/heap"
    "fmt"
)

type Item struct{ d, u int }
type PQ []Item
func (h PQ) Len() int { return len(h) }
func (h PQ) Less(i, j int) bool { return h[i].d < h[j].d }
func (h PQ) Swap(i, j int) { h[i], h[j] = h[j], h[i] }
func (h *PQ) Push(x any) { *h = append(*h, x.(Item)) }
func (h *PQ) Pop() any { old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x }

const INF = int(1e18)

func dijkstra(graph map[int][][2]int, start, n int) []int {
    dist := make([]int, n + 1)
    for i := range dist { dist[i] = INF }
    dist[start] = 0
    pq := &PQ{Item{0, start}}; heap.Init(pq)
    for pq.Len() > 0 {
        cur := heap.Pop(pq).(Item)
        if cur.d > dist[cur.u] { continue }
        for _, e := range graph[cur.u] {
            v, w := e[0], e[1]
            if dist[cur.u] + w < dist[v] {
                dist[v] = dist[cur.u] + w
                heap.Push(pq, Item{dist[v], v})
            }
        }
    }
    return dist
}

func main() {
    g := map[int][][2]int{1:{{2,1},{3,2},{4,1}}, 2:{{3,5},{4,4}}, 3:{{5,1}}, 4:{{3,3},{5,7}}, 5:{}}
    fmt.Println(dijkstra(g, 1, 5))
}
```

### Rust
```rust
use std::collections::BinaryHeap;
use std::cmp::Reverse;

fn dijkstra(graph: &Vec<Vec<(usize, i64)>>, start: usize, n: usize) -> Vec<i64> {
    let mut dist = vec![i64::MAX; n + 1];
    dist[start] = 0;
    let mut pq: BinaryHeap<Reverse<(i64, usize)>> = BinaryHeap::new();
    pq.push(Reverse((0, start)));
    while let Some(Reverse((d, u))) = pq.pop() {
        if d > dist[u] { continue; }
        for &(v, w) in &graph[u] {
            if dist[u] + w < dist[v] {
                dist[v] = dist[u] + w;
                pq.push(Reverse((dist[v], v)));
            }
        }
    }
    dist
}
```

### C#
```csharp
using System;
using System.Collections.Generic;

public class Dij {
    public static int[] Dijkstra(Dictionary<int, List<(int, int)>> graph, int start, int n) {
        int[] dist = new int[n + 1];
        for (int i = 0; i < dist.Length; i++) dist[i] = int.MaxValue;
        dist[start] = 0;
        var pq = new PriorityQueue<int, int>();
        pq.Enqueue(start, 0);
        while (pq.Count > 0) {
            int u = pq.Dequeue();
            if (!graph.ContainsKey(u)) continue;
            foreach (var (v, w) in graph[u]) {
                if (dist[u] + w < dist[v]) {
                    dist[v] = dist[u] + w;
                    pq.Enqueue(v, dist[v]);
                }
            }
        }
        return dist;
    }
}
```

---

## 8. Floyd-Warshall — 모든 쌍 최단 경로 (Python 예시)

```python
def floyd_warshall(graph, n):
    INF = float('inf')
    dist = [[INF] * (n + 1) for _ in range(n + 1)]
    for i in range(1, n + 1): dist[i][i] = 0
    for u in graph:
        for v, w in graph[u]:
            dist[u][v] = w
    
    for k in range(1, n + 1):
        for i in range(1, n + 1):
            for j in range(1, n + 1):
                dist[i][j] = min(dist[i][j], dist[i][k] + dist[k][j])
    return dist
```

---

## 9. 변형 / 응용

### (a) K 번째 최단 경로
Dijkstra 변형 — heap 에 K 개 까지 저장.

### (b) 경로 추적 (Path Reconstruction)
`parent[v]` 배열 추가 — 갱신할 때 `parent[v] = u`. 마지막에 역추적.

### (c) 0-1 BFS
가중치 0/1 → deque 사용. 0 은 앞에, 1 은 뒤에. `O(V+E)`.

### (d) A* (휴리스틱)
Dijkstra + 휴리스틱 함수. 목적지 노드에 빨리 도달.

### (e) 가중치 음수 + 사이클 X (DAG)
Topological Sort + DP.

### (f) 다중 시작점
가상 노드 추가하고 모든 시작점에서 거리 0 으로 연결.

### (g) 두 노드 간 최단 (양방향)
양방향 Dijkstra — 시작 + 끝 동시. 만남 지점.

---

## 10. 함정 / 안티패턴

### 함정 1 — 음수 가중치에 Dijkstra
Dijkstra 는 양수 가중치 전제. 음수 있으면 Bellman-Ford.

### 함정 2 — Python `heapq` 는 min-heap 만
최대 힙 필요하면 `-value` 로 변환.

### 함정 3 — heap 에 같은 노드 여러 번 들어감
정상. `if d > dist[u]: continue` 로 처리. **무시하지 말고 반드시 확인**.

### 함정 4 — 인접 행렬 vs 리스트
V=1000, E=1000 → 인접 리스트 (행렬은 10^6 메모리). V=500, E=10^5 → 행렬 OK.

### 함정 5 — 무방향 vs 방향
무방향이면 `add_edge(u, v); add_edge(v, u)`. 깜빡하면 결과 틀림.

### 함정 6 — INF 비교 오버플로
`dist[u] + w` 가 `INT_MAX` 더하면 오버플로. **`long long` / 또는 `INT_MAX / 2`** 사용.

### 함정 7 — Floyd-Warshall 순서
**k 가 가장 바깥**. 안쪽이면 결과 틀림.

### 함정 8 — 방문 마킹
Dijkstra 에서 `visited` 필요할 수도 / 없을 수도. heap-based 면 `d > dist[u]` 만으로 충분.

---

## 11. 실전 풀이 절차 — 6 단계

1. **그래프 인식** — 노드 / 간선 / 가중치 추출.
2. **알고리즘 선택** — BFS / Dijkstra / Bellman / Floyd.
3. **인접 자료구조** — 리스트 (대부분) / 행렬.
4. **시작점** — 단일 / 다중.
5. **경로 vs 거리** — 거리만 / 경로도 (parent 배열).
6. **결과 추출** — `dist[end]` / `dist 전체` / 역추적.

---

## 12. 대표 문제 (이코테 Ch.09 + Ch.17)

### Q1. 미래 도시 (Ch.09) — Floyd-Warshall

```python
INF = int(1e9)
n, m = map(int, input().split())
graph = [[INF]*(n+1) for _ in range(n+1)]
for i in range(1, n+1): graph[i][i] = 0
for _ in range(m):
    a, b = map(int, input().split())
    graph[a][b] = graph[b][a] = 1
x, k = map(int, input().split())

for ki in range(1, n+1):
    for i in range(1, n+1):
        for j in range(1, n+1):
            graph[i][j] = min(graph[i][j], graph[i][ki] + graph[ki][j])

result = graph[1][k] + graph[k][x]
print(result if result < INF else -1)
```

### Q2. 전보 (Ch.09) — Dijkstra
도시 N 개에 X 부터 메시지 전파. 받은 도시 수 + 마지막 도착 시간.

```python
import heapq
n, m, c = map(int, input().split())
graph = [[] for _ in range(n+1)]
for _ in range(m):
    x, y, z = map(int, input().split())
    graph[x].append((y, z))

INF = int(1e9)
dist = [INF] * (n + 1)
dist[c] = 0
pq = [(0, c)]
while pq:
    d, u = heapq.heappop(pq)
    if d > dist[u]: continue
    for v, w in graph[u]:
        if dist[u] + w < dist[v]:
            dist[v] = dist[u] + w
            heapq.heappush(pq, (dist[v], v))

count = sum(1 for d in dist[1:] if d != INF) - 1
last = max(d for d in dist[1:] if d != INF)
print(count, last)
```

### Q3. 플로이드 (Ch.17, 백준 11404)

```python
INF = int(1e9)
n = int(input())
m = int(input())
g = [[INF]*(n+1) for _ in range(n+1)]
for i in range(1, n+1): g[i][i] = 0
for _ in range(m):
    a, b, c = map(int, input().split())
    g[a][b] = min(g[a][b], c)
for k in range(1, n+1):
    for i in range(1, n+1):
        for j in range(1, n+1):
            g[i][j] = min(g[i][j], g[i][k] + g[k][j])

for i in range(1, n+1):
    print(' '.join(str(g[i][j]) if g[i][j] != INF else '0' for j in range(1, n+1)))
```

### Q4. 정확한 순위 (Ch.17) — Floyd-Warshall
N 명 학생 비교 결과. 정확히 순위 알 수 있는 학생 수.

```python
INF = int(1e9)
n, m = map(int, input().split())
g = [[INF]*(n+1) for _ in range(n+1)]
for i in range(1, n+1): g[i][i] = 0
for _ in range(m):
    a, b = map(int, input().split())
    g[a][b] = 1   # a < b
for k in range(1, n+1):
    for i in range(1, n+1):
        for j in range(1, n+1):
            g[i][j] = min(g[i][j], g[i][k] + g[k][j])

result = 0
for i in range(1, n+1):
    count = sum(1 for j in range(1, n+1) if g[i][j] != INF or g[j][i] != INF)
    if count == n: result += 1
print(result)
```

### Q5. 숨바꼭질 (Ch.17, 백준 6118) — Dijkstra

```python
import heapq
n, m = map(int, input().split())
graph = [[] for _ in range(n+1)]
for _ in range(m):
    a, b = map(int, input().split())
    graph[a].append((b, 1))
    graph[b].append((a, 1))

dist = [float('inf')] * (n + 1)
dist[1] = 0
pq = [(0, 1)]
while pq:
    d, u = heapq.heappop(pq)
    if d > dist[u]: continue
    for v, w in graph[u]:
        if dist[u] + w < dist[v]:
            dist[v] = dist[u] + w
            heapq.heappush(pq, (dist[v], v))

max_d = max(d for d in dist[1:] if d != float('inf'))
node = dist.index(max_d)
count = sum(1 for d in dist if d == max_d)
print(node, max_d, count)
```

---

## 13. 연습 문제 (난이도별)

### 입문
- [백준 1753 최단경로](https://www.acmicpc.net/problem/1753) — 표준 Dijkstra
- [백준 11404 플로이드](https://www.acmicpc.net/problem/11404)
- [백준 1916 최소비용 구하기](https://www.acmicpc.net/problem/1916)

### 중급
- [백준 1238 파티](https://www.acmicpc.net/problem/1238) — 왕복 Dijkstra
- [백준 1504 특정한 최단 경로](https://www.acmicpc.net/problem/1504)
- [백준 5972 택배 배송](https://www.acmicpc.net/problem/5972)

### 고급
- [백준 1854 K번째 최단경로 찾기](https://www.acmicpc.net/problem/1854)
- [백준 11657 타임머신](https://www.acmicpc.net/problem/11657) — Bellman-Ford
- [백준 1865 웜홀](https://www.acmicpc.net/problem/1865) — 음수 사이클 감지

---

## 14. 학습 자료

- 이코테 **Ch.09, 17**
- [동빈나 — 최단 경로](https://www.youtube.com/watch?v=acqm9mM1P6o)
- [CLRS Ch.24-25](https://mitpress.mit.edu/9780262046305/) — Single-source / All-pairs Shortest Paths
- [VisuAlgo — Dijkstra](https://visualgo.net/en/sssp)
- [Floyd-Warshall 애니메이션](https://visualgo.net/en/sssp)

---

## 15. 관련

- [[../graph-theory/graph-theory]] — MST / Union-Find / Topological
- [[../dfs-bfs/dfs-bfs]] — 가중치 없는 최단 = BFS
- [[../greedy/greedy]] — Dijkstra 는 그리디 본질
- [[../algorithm|↑ algorithm 인덱스]]
