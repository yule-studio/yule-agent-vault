---
title: "Union-Find — Disjoint Set Union"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:00:00+09:00
tags:
  - data-structure
  - union-find
  - dsu
---

# Union-Find — Disjoint Set Union (DSU)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Path compression / Union by rank |

**[[advanced|↑ Advanced]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

**여러 집합 — 합치기 / 같은 집합 검사** — 거의 O(1). Kruskal MST / 연결 컴포넌트 / 동적 connectivity.

---

## 2. 동작

```
초기:  {1} {2} {3} {4} {5}

union(1, 2):  {1, 2} {3} {4} {5}
union(3, 4):  {1, 2} {3, 4} {5}
union(1, 3):  {1, 2, 3, 4} {5}

find(2) → root → 1 (또는 group ID)
find(5) → 5

same_group(2, 4) → True (둘 다 root 1)
```

---

## 3. 구현 — Forest

### Naive
```python
parent = list(range(n))

def find(x):
    while parent[x] != x:
        x = parent[x]
    return x

def union(a, b):
    parent[find(a)] = find(b)
```

### Find O(N) worst — Skewed tree

---

## 4. Path Compression

```python
def find(x):
    while parent[x] != x:
        parent[x] = parent[parent[x]]    # halving
        x = parent[x]
    return x

# 또는 recursive:
def find(x):
    if parent[x] != x:
        parent[x] = find(parent[x])
    return parent[x]
```

### 효과
- Find 시 — 노드의 parent 를 root 로
- 다음 find — 짧음

---

## 5. Union by Rank / Size

### Union by Rank
```python
rank = [0] * n

def union(a, b):
    ra, rb = find(a), find(b)
    if ra == rb: return False
    if rank[ra] < rank[rb]: ra, rb = rb, ra
    parent[rb] = ra
    if rank[ra] == rank[rb]: rank[ra] += 1
    return True
```

### Union by Size
```python
size = [1] * n

def union(a, b):
    ra, rb = find(a), find(b)
    if ra == rb: return False
    if size[ra] < size[rb]: ra, rb = rb, ra
    parent[rb] = ra
    size[ra] += size[rb]
    return True
```

### 효과
- 항상 작은 tree 를 큰 tree 의 아래로
- 높이 ≤ log N

---

## 6. 시간 복잡도

| | Naive | Compression | + Rank | Both |
| --- | --- | --- | --- | --- |
| Find | O(N) | amortized O(log N) | O(log N) | O(α(N)) |
| Union | O(N) | O(log N) | O(log N) | O(α(N)) |

### α(N) — Inverse Ackermann
- 모든 실용적인 N — α(N) < 5
- "거의 O(1)"

---

## 7. 응용

### 7.1 Kruskal MST
```python
edges.sort(by_weight)
for u, v, w in edges:
    if union(u, v):
        mst.append((u, v, w))
```

### 7.2 Connected Components (offline)
- 모든 edge — union
- 같은 component — same root

### 7.3 Dynamic Connectivity
- Union, query "same component?"

### 7.4 Cycle Detection (undirected)
- Edge (u, v) — union(u, v) 했는데 이미 같은 group? → cycle

### 7.5 Number of Islands II (LeetCode 305)
- 점진적 land 추가 — union with neighbors

### 7.6 Accounts Merge (LeetCode 721)
- 같은 이메일 — union

### 7.7 Friend Circle / Social network
- 친구 관계 — union

### 7.8 Image segmentation
- 픽셀 — node, 같은 color 인접 — union

---

## 8. Weighted Union-Find

### 정의
- Edge 마다 weight (relation)
- find 시 — root 까지의 weight 누적

### 사용
- "Equations Possible" (a/b = k 같은)
- LeetCode 399 Evaluate Division

```python
weight = [1.0] * n
parent = list(range(n))

def find(x):
    if parent[x] != x:
        old_parent = parent[x]
        parent[x] = find(parent[x])
        weight[x] *= weight[old_parent]
    return parent[x]

def union(a, b, val):    # a/b = val
    ra, rb = find(a), find(b)
    if ra != rb:
        parent[ra] = rb
        weight[ra] = val * weight[b] / weight[a]
```

---

## 9. Persistent / Rollback Union-Find

### Rollback
- Path compression 없이
- Undo 가능

### 사용
- Offline dynamic connectivity
- Divide & Conquer + Union-Find

---

## 10. 함정

### 함정 1 — Path compression + Rank
둘 다 적용 — 최고 성능. 한쪽만 — log N.

### 함정 2 — find(a) 캐시
같은 find 여러 번 호출 — 캐시 / 한 번만.

### 함정 3 — Tree 크기 잘못 추적
union 후 size update 잊음.

### 함정 4 — Disconnected
초기 — 각자 자기 root. n 개 component.

### 함정 5 — Delete / split
표준 DSU — split 안 됨. Link-cut tree.

### 함정 6 — Hash key 의 union
key 가 int 가 아니면 — 매핑 table.

---

## 11. 학습 자료

- CLRS Ch 21
- "Algorithms" Sedgewick — Union-Find
- "Disjoint-Set Forests" — Tarjan

---

## 12. 관련

- [[advanced]] — Hub
- [[../graphs/mst]] — Kruskal
- [[../graphs/graphs]] — Connected components
