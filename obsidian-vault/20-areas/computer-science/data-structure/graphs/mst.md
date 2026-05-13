---
title: "MST — Minimum Spanning Tree"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:20:00+09:00
tags:
  - data-structure
  - graph
  - mst
---

# MST — Minimum Spanning Tree

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Kruskal / Prim / 응용 |

**[[graphs|↑ Graphs]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

**모든 노드 연결 + 가중치 합 최소** 인 spanning tree. V-1 edge. Kruskal / Prim 으로 O(E log V).

---

## 2. Spanning Tree

### 정의
- Connected, undirected graph 의 subset
- 모든 노드 포함
- Cycle 없음 (tree)
- V-1 edge

### MST
- Spanning tree 중 — 가중치 합 가장 작은

---

## 3. Kruskal — Edge 중심

### 알고리즘
1. Edge — 가중치 오름차순 정렬
2. 가중치 작은 edge — 사이클 안 만들면 추가
3. V-1 edge 까지

### 구현 — Union-Find
```python
def kruskal(n, edges):
    edges.sort(key=lambda e: e[2])    # (u, v, w) by w
    parent = list(range(n))
    rank = [0] * n
    
    def find(x):
        while parent[x] != x:
            parent[x] = parent[parent[x]]    # path compression
            x = parent[x]
        return x
    
    def union(a, b):
        a, b = find(a), find(b)
        if a == b: return False
        if rank[a] < rank[b]: a, b = b, a
        parent[b] = a
        if rank[a] == rank[b]: rank[a] += 1
        return True
    
    mst = []
    for u, v, w in edges:
        if union(u, v):
            mst.append((u, v, w))
            if len(mst) == n - 1: break
    return mst
```

### 시간
- Sort — O(E log E)
- Union-Find — O(E α(V))
- 총 — O(E log E) = O(E log V)

자세히 → [[../advanced/union-find]] (TBD)

---

## 4. Prim — Node 중심

### 알고리즘
1. 임의 노드 시작
2. 현재 tree 의 가장 작은 outgoing edge 추가
3. V 노드 까지

### 구현 — Priority Queue
```python
import heapq

def prim(graph, start):
    visited = {start}
    edges = [(w, start, v) for v, w in graph[start]]
    heapq.heapify(edges)
    mst = []
    
    while edges and len(visited) < len(graph):
        w, u, v = heapq.heappop(edges)
        if v in visited: continue
        visited.add(v)
        mst.append((u, v, w))
        for next_v, next_w in graph[v]:
            if next_v not in visited:
                heapq.heappush(edges, (next_w, v, next_v))
    return mst
```

### 시간
- Binary heap — O((V+E) log V)
- Fibonacci heap — O(E + V log V)

---

## 5. Kruskal vs Prim

| | Kruskal | Prim |
| --- | --- | --- |
| **자료구조** | Union-Find | Priority Queue |
| **흐름** | Edge 중심 | Node 중심 |
| **Disconnected** | 처리 (Spanning Forest) | 한 component 만 |
| **Sparse graph** | 좋음 | 좋음 |
| **Dense graph** | 보통 | Adjacency matrix 의 O(V²) 가능 |
| **Implementation** | 짧음 | 약간 길음 |

### 선택
- 작은 graph + edge 적음 — Kruskal
- 큰 dense graph — Prim with array (O(V²))

---

## 6. Cut Property

### 정리
- Cut — V 의 partition (S, V-S)
- S 와 V-S 사이의 가장 가벼운 edge — MST 의 일부

### 사용
- Kruskal — light edge across components → MST
- Prim — light edge from visited to unvisited → MST

---

## 7. Cycle Property

### 정리
- Cycle 의 가장 무거운 edge — MST 의 일부 X

### 사용
- 알고리즘 정확성 증명

---

## 8. Borůvka's Algorithm

### 정의
- 1926 (Otakar Borůvka) — 최초의 MST
- Parallel 친화

### 알고리즘
- 매 phase — 각 component 의 가장 가벼운 outgoing edge 선택
- log V phase

### 시간 — O(E log V)
- Parallel — 매우 빠름

---

## 9. Reverse-delete

### 정의
- 큰 weight edge 부터 — 제거해도 connected 라면 제거

### 시간 — O(E² log V) (느림)

### 의미
- 이론적

---

## 10. 응용

### Network design
- 케이블 / 전선 — 최소 비용 연결

### Cluster analysis
- Single linkage clustering — MST + 큰 edge 제거

### Image segmentation
- 픽셀 graph — MST

### Approximate TSP
- MST → 2 배 한계 — 근사

### Maze generation
- Random MST — 미로

### Network routing
- Multicast tree

---

## 11. Steiner Tree

### 정의
- MST — 모든 노드 포함
- Steiner — 지정된 일부 노드만 + 도와줄 가능 노드 활용

### 시간
- 일반 — NP-hard
- 근사 — 2-approximation (MST 기반)

---

## 12. Minimum Spanning Forest

### 정의
- Disconnected graph — 각 component 의 MST

### Kruskal — 자연 (모든 edge 시도)
- V - components 개 edge

---

## 13. 함정

### 함정 1 — Directed graph 의 MST
일반 MST — undirected only. Directed — Edmonds (Chu-Liu).

### 함정 2 — 중복 weight 의 unique X
같은 weight 의 다른 MST 가능.

### 함정 3 — Disconnected graph
Kruskal — Spanning Forest 반환. Prim — 한 component.

### 함정 4 — Negative weight
OK (Kruskal / Prim 정상).

### 함정 5 — 정렬 시간
Kruskal — 정렬이 dominant. Pre-sorted 시 O(E α(V)).

---

## 14. 학습 자료

- CLRS Ch 23
- "Algorithms" Sedgewick
- "Network Algorithms" — Cormen

---

## 15. 관련

- [[graphs]] — Hub
- [[shortest-path]] — Prim 와 Dijkstra 비교
- [[../advanced/union-find]] (TBD) — Kruskal 의 핵심
- [[../heaps/heaps]] — Prim 의 PQ
