---
title: "Segment Tree — 구간 쿼리 트리"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T16:25:00+09:00
tags:
  - data-structure
  - segment-tree
  - range-query
---

# Segment Tree

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Build / Query / Update / Lazy |

**[[trees|↑ Trees]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

**구간 쿼리 + 점 갱신** 모두 O(log N). 합 / 최댓값 / GCD 등. 경쟁 알고리즘의 핵심.

---

## 2. 동기

### Prefix Sum 의 한계
- Range query — O(1)
- Point update — O(N) (재구축)

### Segment Tree
- 둘 다 O(log N)
- 더 유연 (모든 결합법칙 연산)

---

## 3. 구조

### 정의
- Complete binary tree
- 각 노드 — 구간 [l, r] 의 결과 (합 / max 등)
- Leaf — 단일 원소

### 예 — Sum, arr = [1, 3, 5, 7, 9, 11]
```
                  [0-5]: 36
               /              \
           [0-2]: 9         [3-5]: 27
          /      \         /         \
       [0-1]:4 [2]:5    [3-4]:16   [5]:11
       /  \             /    \
     [0]:1 [1]:3      [3]:7  [4]:9
```

### 크기 — 4N (여유)

---

## 4. Build

```python
def build(arr, tree, node, start, end):
    if start == end:
        tree[node] = arr[start]
        return
    mid = (start + end) // 2
    build(arr, tree, 2*node, start, mid)
    build(arr, tree, 2*node+1, mid+1, end)
    tree[node] = tree[2*node] + tree[2*node+1]
```

### 시간 — O(N)

---

## 5. Range Query

```python
def query(tree, node, start, end, L, R):
    if R < start or end < L:
        return 0    # identity for sum
    if L <= start and end <= R:
        return tree[node]
    mid = (start + end) // 2
    return (query(tree, 2*node, start, mid, L, R) +
            query(tree, 2*node+1, mid+1, end, L, R))
```

### 시간 — O(log N)
- 한 path — log N
- 각 level — O(1)

---

## 6. Point Update

```python
def update(tree, node, start, end, idx, val):
    if start == end:
        tree[node] = val
        return
    mid = (start + end) // 2
    if idx <= mid:
        update(tree, 2*node, start, mid, idx, val)
    else:
        update(tree, 2*node+1, mid+1, end, idx, val)
    tree[node] = tree[2*node] + tree[2*node+1]
```

### 시간 — O(log N)

---

## 7. Lazy Propagation — Range Update

### 문제
- 구간 [L, R] 에 +x 적용
- Naive — O(N)
- Lazy — O(log N)

### 아이디어
- 노드의 lazy[] 배열 — 미루기
- 자식 방문 시 — push down

```python
def update_range(tree, lazy, node, start, end, L, R, val):
    # push down
    if lazy[node]:
        tree[node] += (end - start + 1) * lazy[node]
        if start != end:
            lazy[2*node] += lazy[node]
            lazy[2*node+1] += lazy[node]
        lazy[node] = 0
    
    if R < start or end < L:
        return
    if L <= start and end <= R:
        tree[node] += (end - start + 1) * val
        if start != end:
            lazy[2*node] += val
            lazy[2*node+1] += val
        return
    
    mid = (start + end) // 2
    update_range(tree, lazy, 2*node, start, mid, L, R, val)
    update_range(tree, lazy, 2*node+1, mid+1, end, L, R, val)
    tree[node] = tree[2*node] + tree[2*node+1]
```

---

## 8. 변형 — 다른 연산

| 연산 | 결합법칙 | tree[node] 의미 |
| --- | --- | --- |
| Sum | a+b | 구간 합 |
| Max | max(a, b) | 구간 max |
| Min | min(a, b) | 구간 min |
| GCD | gcd(a, b) | 구간 GCD |
| XOR | a^b | 구간 XOR |
| Product (modular) | a*b | 구간 곱 |

### 조건 — Associative (결합법칙)
- (a op b) op c == a op (b op c)
- 매우 다양 (덧셈 / 곱 / max / min / gcd / and / or / xor / matrix mult)

---

## 9. Persistent Segment Tree

### 정의
- 매 update — 새 트리 (옛 트리 유지)
- 메모리 — O(log N) per update

### 사용
- Functional / 시간 여행
- Range k-th smallest

---

## 10. Iterative Segment Tree

```python
class SegTree:
    def __init__(self, n):
        self.n = n
        self.tree = [0] * (2 * n)
    
    def update(self, i, val):
        i += self.n
        self.tree[i] = val
        while i > 1:
            i //= 2
            self.tree[i] = self.tree[2*i] + self.tree[2*i+1]
    
    def query(self, l, r):
        res = 0
        l += self.n
        r += self.n + 1
        while l < r:
            if l & 1:
                res += self.tree[l]
                l += 1
            if r & 1:
                r -= 1
                res += self.tree[r]
            l //= 2
            r //= 2
        return res
```

### 효과
- 코드 짧음
- Recursion 없음 — 빠름
- Lazy 어려움

---

## 11. Fenwick Tree vs Segment Tree

| | Fenwick | Segment |
| --- | --- | --- |
| **공간** | O(N) | O(4N) |
| **Build** | O(N) | O(N) |
| **Point update** | O(log N) | O(log N) |
| **Range query** | O(log N) | O(log N) |
| **Range update** | O(log N) | O(log N) (lazy) |
| **연산** | Sum / XOR (역연산 필요) | 모든 결합법칙 |
| **코드** | 짧음 | 길음 |

자세히 → [[../advanced/fenwick-tree]] (TBD)

---

## 12. Sparse Table

### 정적 — 갱신 X
- Build O(N log N)
- Query O(1) (RMQ 등 idempotent 연산)

### vs Segment
- 갱신 없으면 — Sparse Table 더 빠름
- 갱신 있으면 — Segment

자세히 → competitive

---

## 13. 응용

### Range sum / max / min — 빈번
### Inversion count (merge sort 대안)
### Persistent — k-th smallest in range
### Lazy — range modification
### 2D segment tree — 행렬 query
### Wavelet tree — k-th in range

---

## 14. 함정

### 함정 1 — 크기 계산
4N 또는 2 × ⌈log N⌉ + 1. 안전하게 4N.

### 함정 2 — 0 vs 1 index
일관성. 보통 — 외부 0-based, 내부 1-based (heap-like).

### 함정 3 — Lazy 의 propagation
Update 전 lazy 처리 필수. 누락 — 잘못된 결과.

### 함정 4 — Range update vs Point update
다른 연산 — 다른 lazy.

### 함정 5 — Off-by-one
[L, R] inclusive vs [L, R) exclusive. 일관성.

### 함정 6 — 큰 N
N = 10^9 — segment tree 안 됨. Sparse / Coordinate compression / Segment Tree Beats / persistent.

---

## 15. 학습 자료

- CP-Algorithms — Segment Tree
- Competitive Programming Handbook (Halim)
- USACO Guide

---

## 16. 관련

- [[trees]] — Hub
- [[../advanced/fenwick-tree]] (TBD) — 비교
- [[../arrays-and-strings/prefix-sum]] — 정적 대안
