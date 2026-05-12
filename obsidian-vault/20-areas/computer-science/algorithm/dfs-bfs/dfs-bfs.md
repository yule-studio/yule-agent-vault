---
title: "DFS / BFS (Depth-First / Breadth-First Search)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T00:00:00+09:00
tags:
  - algorithm
  - dfs
  - bfs
  - graph
---

# DFS / BFS — 깊이/너비 우선 탐색

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 수준 + 11 언어 (이코테 Ch.05/13) |

**[[../algorithm|↑ algorithm 인덱스]]**

---

## 1. 한 줄 정의

**그래프 / 격자를 모든 노드 한 번씩 방문**하는 두 가지 방법.
- **DFS** — 한 갈래를 끝까지 파고든 뒤 되돌아옴 (스택 / 재귀).
- **BFS** — 가까운 이웃부터 층별로 확장 (큐).

---

## 2. DFS/BFS 는 언제 쓰는가? — 시그널 인식

### (a) 문제 유형 키워드

- **"~ 연결되어 있는가? / 몇 개의 그룹이 있는가?"** — 연결 요소 (Connected Components)
- **"최소 거리 / 최단 시간"** + 가중치 없음 → **BFS**
- **"가능한 모든 경로 / 백트래킹"** → **DFS**
- **격자 + 4/8 방향 이동** — `dx/dy` 패턴
- **"섬의 개수 / 미로 / 영역 색칠"** — flood fill 의 표준
- **"위상 정렬 / 순환 검사"** → DFS

### (b) DFS vs BFS 선택

| 차원 | DFS | BFS |
| --- | --- | --- |
| 자료구조 | 스택 / 재귀 | 큐 (`deque`) |
| 메모리 | O(높이) — 적음 | O(너비) — 많음 |
| 최단 경로 | ❌ (모든 경로 탐색 필요) | ✅ (가중치 없을 때 최단 보장) |
| 모든 경로 / 백트래킹 | ✅ | ❌ (적합 X) |
| 순환 검사 / 위상 정렬 | ✅ | △ (Kahn's BFS) |
| 트리 — 부모 / 자식 관계 | ✅ | △ |

### (c) DFS/BFS 와 다른 알고리즘

- **가중치 있는 최단 경로** → BFS 가 아니라 **Dijkstra** ([[../shortest-path/shortest-path]])
- **MST** → BFS 가 아니라 **Kruskal/Prim** ([[../graph-theory/graph-theory]])
- **최단 거리 + 모든 노드** → BFS / Floyd-Warshall

---

## 3. 핵심 직관

### DFS — "미로에서 한 방향으로 갈 데까지 갔다가 돌아옴"
손을 벽에 대고 미로 한 쪽으로 끝까지 들어감 → 막다른 길 → 한 갈래 되돌아옴 → 다른 방향.

### BFS — "물이 한 방울에서 사방으로 퍼지듯"
한 점에서 1 칸 이웃, 다음 2 칸 이웃, 다음 3 칸 이웃을 동시에. 층(layer)별 확장.

---

## 4. 필수 자료 구조 기초 (이코테 Ch.05-1)

### 스택 (Stack)
- LIFO (Last In, First Out).
- Python `list.append() / pop()` 으로 사용 — O(1).
- DFS 의 핵심.

### 큐 (Queue)
- FIFO (First In, First Out).
- Python `collections.deque` 의 `append() / popleft()` 사용 — O(1).
- BFS 의 핵심. (`list.pop(0)` 은 O(N) — 금지!)

### 재귀 (Recursion)
- 함수가 자기 자신 호출. 종료 조건 (base case) 필수.
- Python 기본 재귀 한도 1000 → 깊은 재귀에서 `sys.setrecursionlimit(10**6)`.
- 스택 오버플로 위험 — 큰 입력에서 반복문 (스택) 으로 변환 권장.

---

## 5. 동작 원리 — 예시로 깊이

### DFS 예시 — 그래프 노드 7 개

```
그래프 (인접 리스트):
  1: [2, 3, 8]
  2: [1, 7]
  3: [1, 4, 5]
  4: [3, 5]
  5: [3, 4]
  6: [7]
  7: [2, 6, 8]
  8: [1, 7]

DFS 시작 = 1

Step 1: 방문 [1].             스택: [1]
Step 2: 1 → 2 방문. 스택: [1, 2]. 방문 [1, 2]
Step 3: 2 → 7 방문. 스택: [1, 2, 7]. 방문 [1, 2, 7]
Step 4: 7 → 6 방문. 스택: [1, 2, 7, 6]. 방문 [1, 2, 7, 6]
Step 5: 6 의 이웃 7 = 방문됨. 6 빠짐. 스택: [1, 2, 7]
Step 6: 7 → 8 방문. 스택: [1, 2, 7, 8]. 방문 [..., 8]
Step 7: 8 의 이웃 1 = 방문됨. 8 빠짐.
Step 8: 7 이웃 모두 방문됨. 7 빠짐. 스택: [1, 2]
Step 9: 2 이웃 모두 방문됨. 2 빠짐. 스택: [1]
Step 10: 1 → 3 방문. 스택: [1, 3]
Step 11: 3 → 4 방문. ...
...

DFS 순서: 1 → 2 → 7 → 6 → 8 → 3 → 4 → 5
```

### BFS 예시 — 같은 그래프

```
BFS 시작 = 1

Step 1: 1 큐에. 방문 [1]. 큐: [1]
Step 2: 1 빼고 이웃 [2, 3, 8] 큐에. 방문 [1, 2, 3, 8]. 큐: [2, 3, 8]
Step 3: 2 빼고 이웃 [7] 큐에 (1 이미 방문). 방문 [..., 7]. 큐: [3, 8, 7]
Step 4: 3 빼고 이웃 [4, 5] 큐에. 방문 [..., 4, 5]. 큐: [8, 7, 4, 5]
Step 5: 8 빼고. 1, 7 이미 방문. 큐: [7, 4, 5]
Step 6: 7 빼고 [6] 큐에. 방문 [..., 6]. 큐: [4, 5, 6]
...

BFS 순서: 1 → 2 → 3 → 8 → 7 → 4 → 5 → 6
```

층:
- Level 0: {1}
- Level 1: {2, 3, 8}
- Level 2: {7, 4, 5}
- Level 3: {6}

BFS 가 **층 단위로 확장** = 가중치 없는 그래프의 **최단 거리** 자동 계산.

---

## 6. 복잡도

| 변형 | 시간 | 공간 |
| --- | --- | --- |
| 인접 리스트 그래프 | `O(V + E)` | `O(V)` (visited + 큐/스택) |
| 인접 행렬 그래프 | `O(V^2)` | `O(V^2)` |
| 격자 N×M | `O(N×M)` | `O(N×M)` |
| 백트래킹 (DFS) | `O(분기^깊이)` | `O(깊이)` |

V = 노드, E = 간선. **인접 리스트가 거의 항상 효율**.

---

## 7. 의사 코드

### DFS (재귀)
```
function DFS(node, visited):
    visited[node] ← True
    process(node)
    for neighbor in graph[node]:
        if not visited[neighbor]:
            DFS(neighbor, visited)
```

### DFS (반복 — 스택)
```
function DFS(start):
    stack ← [start]
    visited ← {start}
    while stack not empty:
        node ← stack.pop()
        process(node)
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                stack.push(neighbor)
```

### BFS
```
function BFS(start):
    queue ← deque([start])
    visited ← {start}
    while queue not empty:
        node ← queue.popleft()
        process(node)
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)
```

---

## 8. 언어별 구현 — DFS + BFS (11 언어)

### Python 3
```python
from collections import deque

graph = {
    1: [2, 3, 8], 2: [1, 7], 3: [1, 4, 5],
    4: [3, 5], 5: [3, 4], 6: [7], 7: [2, 6, 8], 8: [1, 7]
}

# DFS — 재귀
def dfs(node, visited):
    visited[node] = True
    print(node, end=' ')
    for nxt in graph[node]:
        if not visited[nxt]:
            dfs(nxt, visited)

# DFS — 반복
def dfs_iter(start):
    stack = [start]
    visited = {start}
    while stack:
        node = stack.pop()
        print(node, end=' ')
        for nxt in graph[node]:
            if nxt not in visited:
                visited.add(nxt)
                stack.append(nxt)

# BFS
def bfs(start):
    queue = deque([start])
    visited = {start}
    while queue:
        node = queue.popleft()
        print(node, end=' ')
        for nxt in graph[node]:
            if nxt not in visited:
                visited.add(nxt)
                queue.append(nxt)

dfs(1, {i: False for i in range(1, 9)})   # 1 2 7 6 8 3 4 5
print()
bfs(1)                                     # 1 2 3 8 7 4 5 6
```

### Java
```java
import java.util.*;

public class DfsBfs {
    static Map<Integer, List<Integer>> graph = Map.of(
        1, List.of(2,3,8), 2, List.of(1,7), 3, List.of(1,4,5),
        4, List.of(3,5), 5, List.of(3,4), 6, List.of(7), 7, List.of(2,6,8), 8, List.of(1,7));

    static void dfs(int node, boolean[] visited) {
        visited[node] = true;
        System.out.print(node + " ");
        for (int nxt : graph.get(node))
            if (!visited[nxt]) dfs(nxt, visited);
    }

    static void bfs(int start) {
        Queue<Integer> q = new ArrayDeque<>();
        boolean[] visited = new boolean[9];
        q.offer(start); visited[start] = true;
        while (!q.isEmpty()) {
            int node = q.poll();
            System.out.print(node + " ");
            for (int nxt : graph.get(node)) {
                if (!visited[nxt]) {
                    visited[nxt] = true;
                    q.offer(nxt);
                }
            }
        }
    }

    public static void main(String[] args) {
        dfs(1, new boolean[9]); System.out.println();
        bfs(1); System.out.println();
    }
}
```

### C++
```cpp
#include <bits/stdc++.h>
using namespace std;

map<int, vector<int>> graph = {
    {1,{2,3,8}}, {2,{1,7}}, {3,{1,4,5}}, {4,{3,5}},
    {5,{3,4}}, {6,{7}}, {7,{2,6,8}}, {8,{1,7}}};

void dfs(int node, vector<bool>& visited) {
    visited[node] = true;
    cout << node << ' ';
    for (int nxt : graph[node])
        if (!visited[nxt]) dfs(nxt, visited);
}

void bfs(int start) {
    queue<int> q; vector<bool> visited(9, false);
    q.push(start); visited[start] = true;
    while (!q.empty()) {
        int node = q.front(); q.pop();
        cout << node << ' ';
        for (int nxt : graph[node]) {
            if (!visited[nxt]) {
                visited[nxt] = true;
                q.push(nxt);
            }
        }
    }
}

int main() {
    vector<bool> v(9, false);
    dfs(1, v); cout << endl;
    bfs(1); cout << endl;
}
```

### C
```c
#include <stdio.h>
#include <stdbool.h>

int graph[9][9] = {0};        // 인접 행렬
int adj_count[9] = {0};       // 각 노드의 인접 개수

void add_edge(int a, int b) {
    graph[a][adj_count[a]++] = b;
    graph[b][adj_count[b]++] = a;
}

bool visited[9];

void dfs(int node) {
    visited[node] = true;
    printf("%d ", node);
    for (int i = 0; i < adj_count[node]; i++) {
        int nxt = graph[node][i];
        if (!visited[nxt]) dfs(nxt);
    }
}

void bfs(int start) {
    int queue[100], front = 0, rear = 0;
    queue[rear++] = start;
    bool seen[9] = {false};
    seen[start] = true;
    while (front < rear) {
        int node = queue[front++];
        printf("%d ", node);
        for (int i = 0; i < adj_count[node]; i++) {
            int nxt = graph[node][i];
            if (!seen[nxt]) {
                seen[nxt] = true;
                queue[rear++] = nxt;
            }
        }
    }
}

int main() {
    add_edge(1,2); add_edge(1,3); add_edge(1,8);
    add_edge(2,7); add_edge(3,4); add_edge(3,5);
    add_edge(4,5); add_edge(6,7); add_edge(7,8);
    dfs(1); printf("\n");
    bfs(1); printf("\n");
}
```

### JavaScript (Node.js)
```javascript
const graph = {
    1:[2,3,8], 2:[1,7], 3:[1,4,5], 4:[3,5],
    5:[3,4], 6:[7], 7:[2,6,8], 8:[1,7]
};

function dfs(node, visited) {
    visited[node] = true;
    process.stdout.write(node + ' ');
    for (const nxt of graph[node])
        if (!visited[nxt]) dfs(nxt, visited);
}

function bfs(start) {
    const queue = [start];
    const visited = {[start]: true};
    while (queue.length) {
        const node = queue.shift();        // 작은 입력 OK. 큰 입력은 deque-like 구조 필요
        process.stdout.write(node + ' ');
        for (const nxt of graph[node]) {
            if (!visited[nxt]) {
                visited[nxt] = true;
                queue.push(nxt);
            }
        }
    }
}

dfs(1, {}); console.log();
bfs(1); console.log();
```

### TypeScript
```typescript
const graph: Record<number, number[]> = {
    1:[2,3,8], 2:[1,7], 3:[1,4,5], 4:[3,5],
    5:[3,4], 6:[7], 7:[2,6,8], 8:[1,7]
};

function dfs(node: number, visited: Record<number, boolean>): void {
    visited[node] = true;
    process.stdout.write(`${node} `);
    for (const nxt of graph[node])
        if (!visited[nxt]) dfs(nxt, visited);
}

function bfs(start: number): void {
    const queue: number[] = [start];
    const visited: Record<number, boolean> = {[start]: true};
    while (queue.length) {
        const node = queue.shift()!;
        process.stdout.write(`${node} `);
        for (const nxt of graph[node]) {
            if (!visited[nxt]) { visited[nxt] = true; queue.push(nxt); }
        }
    }
}
```

### Kotlin
```kotlin
import java.util.ArrayDeque

val graph = mapOf(
    1 to listOf(2,3,8), 2 to listOf(1,7), 3 to listOf(1,4,5),
    4 to listOf(3,5), 5 to listOf(3,4), 6 to listOf(7),
    7 to listOf(2,6,8), 8 to listOf(1,7))

fun dfs(node: Int, visited: BooleanArray) {
    visited[node] = true
    print("$node ")
    for (nxt in graph[node]!!) if (!visited[nxt]) dfs(nxt, visited)
}

fun bfs(start: Int) {
    val q = ArrayDeque<Int>()
    val visited = BooleanArray(9)
    q.offer(start); visited[start] = true
    while (q.isNotEmpty()) {
        val node = q.poll()
        print("$node ")
        for (nxt in graph[node]!!) {
            if (!visited[nxt]) { visited[nxt] = true; q.offer(nxt) }
        }
    }
}

fun main() {
    dfs(1, BooleanArray(9)); println()
    bfs(1); println()
}
```

### Swift
```swift
let graph: [Int: [Int]] = [
    1:[2,3,8], 2:[1,7], 3:[1,4,5], 4:[3,5],
    5:[3,4], 6:[7], 7:[2,6,8], 8:[1,7]
]

func dfs(_ node: Int, _ visited: inout [Bool]) {
    visited[node] = true
    print(node, terminator: " ")
    for nxt in graph[node] ?? [] where !visited[nxt] { dfs(nxt, &visited) }
}

func bfs(_ start: Int) {
    var queue = [start]
    var visited = Array(repeating: false, count: 9)
    visited[start] = true
    while !queue.isEmpty {
        let node = queue.removeFirst()
        print(node, terminator: " ")
        for nxt in graph[node] ?? [] where !visited[nxt] {
            visited[nxt] = true; queue.append(nxt)
        }
    }
}

var v = Array(repeating: false, count: 9)
dfs(1, &v); print()
bfs(1); print()
```

### Go
```go
package main

import (
    "container/list"
    "fmt"
)

var graph = map[int][]int{
    1:{2,3,8}, 2:{1,7}, 3:{1,4,5}, 4:{3,5},
    5:{3,4}, 6:{7}, 7:{2,6,8}, 8:{1,7},
}

func dfs(node int, visited []bool) {
    visited[node] = true
    fmt.Printf("%d ", node)
    for _, nxt := range graph[node] {
        if !visited[nxt] { dfs(nxt, visited) }
    }
}

func bfs(start int) {
    q := list.New()
    q.PushBack(start)
    visited := make([]bool, 9)
    visited[start] = true
    for q.Len() > 0 {
        front := q.Front()
        node := front.Value.(int)
        q.Remove(front)
        fmt.Printf("%d ", node)
        for _, nxt := range graph[node] {
            if !visited[nxt] {
                visited[nxt] = true
                q.PushBack(nxt)
            }
        }
    }
}

func main() {
    dfs(1, make([]bool, 9)); fmt.Println()
    bfs(1); fmt.Println()
}
```

### Rust
```rust
use std::collections::{HashMap, VecDeque};

fn dfs(node: i32, graph: &HashMap<i32, Vec<i32>>, visited: &mut Vec<bool>) {
    visited[node as usize] = true;
    print!("{} ", node);
    for &nxt in &graph[&node] {
        if !visited[nxt as usize] { dfs(nxt, graph, visited); }
    }
}

fn bfs(start: i32, graph: &HashMap<i32, Vec<i32>>) {
    let mut queue: VecDeque<i32> = VecDeque::from([start]);
    let mut visited = vec![false; 9];
    visited[start as usize] = true;
    while let Some(node) = queue.pop_front() {
        print!("{} ", node);
        for &nxt in &graph[&node] {
            if !visited[nxt as usize] {
                visited[nxt as usize] = true;
                queue.push_back(nxt);
            }
        }
    }
}

fn main() {
    let g: HashMap<i32, Vec<i32>> = [
        (1,vec![2,3,8]),(2,vec![1,7]),(3,vec![1,4,5]),(4,vec![3,5]),
        (5,vec![3,4]),(6,vec![7]),(7,vec![2,6,8]),(8,vec![1,7])
    ].into();
    let mut v = vec![false; 9];
    dfs(1, &g, &mut v); println!();
    bfs(1, &g); println!();
}
```

### C#
```csharp
using System;
using System.Collections.Generic;

public class DfsBfs {
    static Dictionary<int, List<int>> graph = new() {
        {1,new(){2,3,8}}, {2,new(){1,7}}, {3,new(){1,4,5}}, {4,new(){3,5}},
        {5,new(){3,4}}, {6,new(){7}}, {7,new(){2,6,8}}, {8,new(){1,7}}
    };

    static void Dfs(int node, bool[] visited) {
        visited[node] = true;
        Console.Write(node + " ");
        foreach (var nxt in graph[node])
            if (!visited[nxt]) Dfs(nxt, visited);
    }

    static void Bfs(int start) {
        var q = new Queue<int>();
        var visited = new bool[9];
        q.Enqueue(start); visited[start] = true;
        while (q.Count > 0) {
            int node = q.Dequeue();
            Console.Write(node + " ");
            foreach (var nxt in graph[node]) {
                if (!visited[nxt]) { visited[nxt] = true; q.Enqueue(nxt); }
            }
        }
    }

    public static void Main() {
        Dfs(1, new bool[9]); Console.WriteLine();
        Bfs(1); Console.WriteLine();
    }
}
```

---

## 9. 변형 / 응용

### (a) 격자 BFS — 최단 거리

```python
from collections import deque
def bfs_grid(grid, start):
    n, m = len(grid), len(grid[0])
    dist = [[-1]*m for _ in range(n)]
    sx, sy = start
    dist[sx][sy] = 0
    q = deque([start])
    dx, dy = [-1,1,0,0], [0,0,-1,1]
    while q:
        x, y = q.popleft()
        for d in range(4):
            nx, ny = x+dx[d], y+dy[d]
            if 0<=nx<n and 0<=ny<m and grid[nx][ny]==1 and dist[nx][ny]==-1:
                dist[nx][ny] = dist[x][y] + 1
                q.append((nx, ny))
    return dist
```

### (b) Flood Fill — 영역 색칠
"음료수 얼려 먹기" (이코테 Ch.05) 의 표준. DFS / BFS 모두 OK.

### (c) 백트래킹 (DFS + 가지치기)
조합 / 순열 / N-Queens. 가능한 모든 경우 시도 + 조건 불만족 시 즉시 되돌림.

### (d) 위상 정렬 (Kahn's BFS / DFS)
DAG 에서 선행 관계 순서로 정렬. [[../graph-theory/graph-theory]] 참조.

### (e) 이분 그래프 판정
BFS / DFS 로 색 칠하기. 인접 노드가 같은 색이면 X.

### (f) 0-1 BFS (Deque)
가중치 0 / 1 인 그래프에서 최단. 0 은 앞에, 1 은 뒤에 deque 삽입.

### (g) 다중 시작점 BFS
시작점이 여러 개 (예: 토마토 익기). 모든 시작점을 큐에 한 번에 넣고 BFS.

---

## 10. 함정 / 안티패턴

### 함정 1 — `visited` 표시 시점
**큐에 넣는 시점에 표시**해야 중복 방문 방지:

```python
# 잘못 — 같은 노드 여러 번 큐 입력
while q:
    node = q.popleft()
    visited[node] = True   # 너무 늦음
    for nxt in graph[node]:
        if not visited[nxt]:
            q.append(nxt)

# 옳음
while q:
    node = q.popleft()
    for nxt in graph[node]:
        if not visited[nxt]:
            visited[nxt] = True   # 큐에 넣자마자
            q.append(nxt)
```

### 함정 2 — `list.pop(0)` 사용
O(N) — 큰 입력에서 TLE. **반드시 `deque`**.

### 함정 3 — Python 재귀 깊이
기본 1000. DFS 깊이가 10^4 ~ 10^6 일 가능성:
```python
import sys
sys.setrecursionlimit(10**6)
```

### 함정 4 — 인접 리스트 vs 인접 행렬
- V = 10^5, E = 10^5 → 인접 리스트 (행렬은 10^10 = 메모리 폭발).
- V = 500, 모든 쌍 → 행렬 OK.

### 함정 5 — 무방향 vs 방향 그래프
무방향이면 양쪽 다 `add_edge(a, b); add_edge(b, a)`. 방향이면 한쪽만.

### 함정 6 — 시작점이 여러 개
다중 시작점 BFS — 모두 큐에 넣고 시작. 거리 0 으로 초기화.

### 함정 7 — BFS 거리 표시
`dist[next] = dist[current] + 1` — 큐에 넣을 때 동시에 갱신.

### 함정 8 — DFS 가지치기 빠짐
백트래킹 / 순열 → 조건 어긋나면 즉시 `return`. 가지치기 안 하면 TLE.

---

## 11. 실전 풀이 절차 — 7 단계

1. **그래프 / 격자 인식** — 노드 / 간선 / 격자 칸으로 변환.
2. **DFS vs BFS 결정** — 최단 거리 → BFS, 모든 경로 → DFS.
3. **자료 구조** — 인접 리스트 (대부분) / `deque` 큐 / `visited`.
4. **시작 조건** — 한 곳 / 여러 곳 / 외곽.
5. **이웃 정의** — `dx/dy` / `graph[node]` / 특수 이동 규칙.
6. **종료 조건** — 큐 비면 / 도달하면 / 모두 방문하면.
7. **결과 추출** — 거리 / 카운트 / 경로 추적 (parent 배열).

---

## 12. 대표 문제 (이코테 Ch.05 + Ch.13)

### Q1. 음료수 얼려 먹기 (Ch.05) — DFS Flood Fill
N×M 격자, 0 은 얼음, 1 은 칸막이. 연결된 0 그룹 개수.

```python
n, m = map(int, input().split())
grid = [list(map(int, input())) for _ in range(n)]

def dfs(x, y):
    if x < 0 or x >= n or y < 0 or y >= m: return False
    if grid[x][y] == 0:
        grid[x][y] = 1
        dfs(x-1, y); dfs(x+1, y); dfs(x, y-1); dfs(x, y+1)
        return True
    return False

count = 0
for i in range(n):
    for j in range(m):
        if dfs(i, j): count += 1
print(count)
```

### Q2. 미로 탈출 (Ch.05) — BFS 최단 경로
N×M 미로. 1 = 길, 0 = 벽. (0,0) → (N-1,M-1) 최단 칸 수.

```python
from collections import deque
n, m = map(int, input().split())
grid = [list(map(int, input())) for _ in range(n)]
dx, dy = [-1,1,0,0], [0,0,-1,1]

def bfs():
    q = deque([(0, 0)])
    while q:
        x, y = q.popleft()
        for d in range(4):
            nx, ny = x+dx[d], y+dy[d]
            if 0<=nx<n and 0<=ny<m and grid[nx][ny] == 1:
                grid[nx][ny] = grid[x][y] + 1
                q.append((nx, ny))
    return grid[n-1][m-1]

print(bfs())
```

### Q3. 특정 거리의 도시 찾기 (Ch.13) — 다중 BFS

```python
from collections import deque
n, m, k, x = map(int, input().split())
graph = [[] for _ in range(n+1)]
for _ in range(m):
    a, b = map(int, input().split())
    graph[a].append(b)

dist = [-1] * (n+1); dist[x] = 0
q = deque([x])
while q:
    now = q.popleft()
    for nxt in graph[now]:
        if dist[nxt] == -1:
            dist[nxt] = dist[now] + 1
            q.append(nxt)

result = [i for i in range(1, n+1) if dist[i] == k]
print('\n'.join(map(str, result)) if result else -1)
```

### Q4. 연구소 (Ch.13) — DFS/BFS + 완전 탐색
빈 칸 3 개에 벽 → 바이러스 확산 → 안전 영역 최대.

```python
from itertools import combinations
from collections import deque

def spread(grid):
    n, m = len(grid), len(grid[0])
    g = [row[:] for row in grid]
    q = deque((i, j) for i in range(n) for j in range(m) if g[i][j] == 2)
    dx, dy = [-1,1,0,0], [0,0,-1,1]
    while q:
        x, y = q.popleft()
        for d in range(4):
            nx, ny = x+dx[d], y+dy[d]
            if 0<=nx<n and 0<=ny<m and g[nx][ny] == 0:
                g[nx][ny] = 2
                q.append((nx, ny))
    return sum(row.count(0) for row in g)

n, m = map(int, input().split())
grid = [list(map(int, input().split())) for _ in range(n)]
empty = [(i, j) for i in range(n) for j in range(m) if grid[i][j] == 0]
best = 0
for walls in combinations(empty, 3):
    g = [row[:] for row in grid]
    for x, y in walls:
        g[x][y] = 1
    best = max(best, spread(g))
print(best)
```

### Q5. 경쟁적 전염 (Ch.13) — 다중 BFS + 정렬

타입 번호가 작은 바이러스부터 확산.

```python
from collections import deque
n, k = map(int, input().split())
grid = [list(map(int, input().split())) for _ in range(n)]
s, x, y = map(int, input().split())

viruses = []
for i in range(n):
    for j in range(n):
        if grid[i][j] > 0:
            viruses.append((grid[i][j], 0, i, j))
viruses.sort()
q = deque(viruses)
dx, dy = [-1,1,0,0], [0,0,-1,1]

while q:
    v, t, i, j = q.popleft()
    if t == s: break
    for d in range(4):
        ni, nj = i+dx[d], j+dy[d]
        if 0<=ni<n and 0<=nj<n and grid[ni][nj] == 0:
            grid[ni][nj] = v
            q.append((v, t+1, ni, nj))

print(grid[x-1][y-1])
```

---

## 13. 연습 문제 (난이도별)

### 입문
- [백준 1260 DFS와 BFS](https://www.acmicpc.net/problem/1260) — 표준 입문
- [백준 2606 바이러스](https://www.acmicpc.net/problem/2606) — 연결 요소
- [백준 11724 연결 요소의 개수](https://www.acmicpc.net/problem/11724)

### 중급
- [백준 7569 토마토 (3D)](https://www.acmicpc.net/problem/7569) — 3차원 BFS
- [백준 1697 숨바꼭질](https://www.acmicpc.net/problem/1697) — 1D BFS
- [백준 2178 미로 탐색](https://www.acmicpc.net/problem/2178) — 격자 BFS
- [백준 14502 연구소](https://www.acmicpc.net/problem/14502) — 위 Q4

### 고급 (백트래킹)
- [백준 14888 연산자 끼워넣기](https://www.acmicpc.net/problem/14888)
- [백준 9663 N-Queen](https://www.acmicpc.net/problem/9663) — 백트래킹 표준
- [백준 1759 암호 만들기](https://www.acmicpc.net/problem/1759)

### 카카오 / 삼성 기출
- [Programmers — 단어 변환](https://programmers.co.kr/learn/courses/30/lessons/43163)
- [Programmers — 네트워크](https://programmers.co.kr/learn/courses/30/lessons/43162)
- [백준 16236 아기 상어](https://www.acmicpc.net/problem/16236) — BFS + 시뮬레이션

---

## 14. 학습 자료

- 이코테 **Ch.05, 13** — 한국어 표준 책
- [동빈나 — DFS/BFS 영상](https://www.youtube.com/watch?v=7C9RgOcvkvo) (무료)
- [CLRS Ch.22](https://mitpress.mit.edu/9780262046305/) — Elementary Graph Algorithms
- [VisuAlgo — DFS/BFS](https://visualgo.net/en/dfsbfs) — 시각화
- [Algorithm Visualizer](https://algorithm-visualizer.org/brute-force/depth-first-search)

---

## 15. 관련

- [[../implementation/implementation]] — 격자 dx/dy 패턴 공유
- [[../shortest-path/shortest-path]] — 가중치 있을 때
- [[../graph-theory/graph-theory]] — 위상 정렬 / 서로소 집합
