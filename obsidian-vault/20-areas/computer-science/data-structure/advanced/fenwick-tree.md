---
title: "Fenwick Tree — Binary Indexed Tree"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:05:00+09:00
tags:
  - data-structure
  - fenwick
  - bit
  - prefix-sum
---

# Fenwick Tree (BIT)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Update / Query O(log N) |

**[[advanced|↑ Advanced]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

**점 갱신 + 구간 합** 모두 O(log N). Segment Tree 보다 짧고 메모리 작음. 1989 Peter Fenwick.

---

## 2. 구조

```
arr (1-base):     [_, a1, a2, a3, a4, a5, a6, a7, a8]
tree (BIT):       [_, t1, t2, t3, t4, t5, t6, t7, t8]

각 t[i] — 일부 구간 합 (i 의 lowest set bit 만큼)

t[1] = a1
t[2] = a1 + a2
t[3] = a3
t[4] = a1 + a2 + a3 + a4
t[5] = a5
t[6] = a5 + a6
t[7] = a7
t[8] = a1 + ... + a8
```

### Lowest Set Bit (lowbit)
```
lowbit(i) = i & (-i)
```

### Index 관계
- t[i] — 길이 lowbit(i) 의 구간

---

## 3. 연산

### Update — point
```python
def update(tree, i, delta):
    while i <= n:
        tree[i] += delta
        i += i & (-i)        # 다음 노드
```

### Query — prefix sum [1, i]
```python
def query(tree, i):
    s = 0
    while i > 0:
        s += tree[i]
        i -= i & (-i)        # 이전 노드
    return s
```

### Range Query [l, r]
```python
def range_query(tree, l, r):
    return query(tree, r) - query(tree, l - 1)
```

---

## 4. 시간 / 공간

| | 시간 |
| --- | --- |
| Build | O(N) (또는 N log N naive) |
| Point Update | O(log N) |
| Prefix Query | O(log N) |
| Range Query | O(log N) |

### 공간 — O(N)

---

## 5. Build

### Naive — O(N log N)
```python
tree = [0] * (n + 1)
for i in range(1, n + 1):
    update(tree, i, arr[i - 1])
```

### Optimal — O(N)
```python
tree = [0] + arr[:]    # 1-base
for i in range(1, n + 1):
    j = i + (i & -i)
    if j <= n:
        tree[j] += tree[i]
```

---

## 6. 구현

```python
class BIT:
    def __init__(self, n):
        self.n = n
        self.tree = [0] * (n + 1)
    
    def update(self, i, delta):
        while i <= self.n:
            self.tree[i] += delta
            i += i & -i
    
    def query(self, i):
        s = 0
        while i > 0:
            s += self.tree[i]
            i -= i & -i
        return s
    
    def range_query(self, l, r):
        return self.query(r) - self.query(l - 1)
```

---

## 7. Fenwick vs Segment

| | Fenwick | Segment |
| --- | --- | --- |
| **공간** | N | 4N |
| **코드 짧음** | Yes | No |
| **Point update + Range query** | O(log N) | O(log N) |
| **Range update + Point query** | O(log N) (변형) | O(log N) |
| **Range update + Range query** | O(log N) (2 BIT) | O(log N) (lazy) |
| **연산** | 역연산 필요 (sum / XOR) | 모든 결합법칙 |

### 선택
- Sum / XOR — Fenwick 권장
- Max / Min / 복잡 — Segment

---

## 8. 변형 — Range Update + Point Query

### Difference array idea
```python
def range_update(bit, l, r, delta):
    update(bit, l, delta)
    update(bit, r + 1, -delta)

def point_query(bit, i):
    return query(bit, i)    # prefix sum of diff
```

---

## 9. 2D Fenwick

```python
class BIT2D:
    def __init__(self, m, n):
        self.m, self.n = m, n
        self.tree = [[0] * (n + 1) for _ in range(m + 1)]
    
    def update(self, x, y, delta):
        i = x
        while i <= self.m:
            j = y
            while j <= self.n:
                self.tree[i][j] += delta
                j += j & -j
            i += i & -i
    
    def query(self, x, y):
        s = 0
        i = x
        while i > 0:
            j = y
            while j > 0:
                s += self.tree[i][j]
                j -= j & -j
            i -= i & -i
        return s
```

### 시간 — O(log M × log N)

---

## 10. 응용

### 10.1 Inversion count
- Mergesort 의 대안
- O(N log N) — BIT
```python
def inversions(arr):
    sorted_arr = sorted(set(arr))
    rank = {v: i + 1 for i, v in enumerate(sorted_arr)}
    bit = BIT(len(rank))
    count = 0
    for x in reversed(arr):
        count += bit.query(rank[x] - 1)
        bit.update(rank[x], 1)
    return count
```

### 10.2 Range sum with point update
- Streaming 데이터

### 10.3 Order statistics
- "K-th smallest in [l, r]" — Merge Sort Tree / Wavelet
- BIT + binary search

### 10.4 Histogram queries
- 각 bin — BIT
- Sliding window

### 10.5 2D 누적 합 with update
- 게임 / 이미지 분석

---

## 11. 함정

### 함정 1 — 1 vs 0 base
BIT 는 항상 1-base. 외부 0-base 와 일관성.

### 함정 2 — lowbit 의 0 처리
i = 0 — 무한 loop (lowbit 0). while i > 0 / while i <= n.

### 함정 3 — Long 의 overflow
Sum 큰 값 — int → long.

### 함정 4 — Coordinate compression
큰 값 — index 로 매핑 (sorted + rank).

### 함정 5 — Range update + Range query
2 BIT 필요 — 복잡. Segment Tree 가 더 단순.

### 함정 6 — Min / Max
역연산 X — BIT 직접 X. Segment Tree.

---

## 12. 학습 자료

- "A New Data Structure for Cumulative Frequency Tables" — Peter Fenwick (1994)
- CP-Algorithms — Fenwick Tree
- USACO Guide

---

## 13. 관련

- [[advanced]] — Hub
- [[../trees/segment-tree]] — 비교
- [[../arrays-and-strings/prefix-sum]] — 정적 대안
