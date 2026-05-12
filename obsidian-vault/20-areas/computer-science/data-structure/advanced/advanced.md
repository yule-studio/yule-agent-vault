---
title: "고급 자료구조 (Advanced Data Structures)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T09:30:00+09:00
tags:
  - data-structure
  - advanced
  - union-find
  - segment-tree
  - fenwick-tree
  - bloom-filter
  - skip-list
---

# 고급 자료구조 (Advanced Data Structures)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 수준 |

**[[../data-structure|↑ data-structure]]** · **[[../../computer-science|↑↑ computer-science]]**

> 이 노트는 **5 개의 핵심 고급 자료구조** — Union-Find, Segment Tree, Fenwick Tree,
> Bloom Filter, Skip List — 를 함께 다룬다. 각 구조는 별개의 깊은 영역이지만,
> 코딩 테스트 & 시스템 설계에서 함께 자주 등장한다.

---

## 1. 한 줄 정의

- **Union-Find (Disjoint Set Union, DSU)** — N 원소의 분할 (partition) 을 유지.
  `union(a,b)`, `find(a)` 거의 O(1).
- **Segment Tree** — 배열의 **구간 질의 / 갱신** O(log N).
- **Fenwick Tree (Binary Indexed Tree)** — 누적합 구간 질의 / 점 갱신 O(log N).
  Segment Tree 의 간략화.
- **Bloom Filter** — **확률적** 멤버십 — 가짜 양성 가능, 가짜 음성 없음.
- **Skip List** — 확률적 균형 트리 대체 — O(log N) 평균. Redis Sorted Set.

---

## 2. Union-Find (DSU)

### 2.1 개념

N 개 원소를 **disjoint set** 으로 관리. 두 연산:
- `find(x)` — x 가 속한 집합의 대표 (root)
- `union(a, b)` — a, b 가 속한 집합 병합

### 2.2 두 가지 최적화

**경로 압축 (Path Compression)** — find 시 모든 노드를 root 직속 자식으로.
**Union by Rank / Size** — 작은 트리를 큰 트리에 붙임.

두 최적화 동시 적용 시 amortized **O(α(N))** — Inverse Ackermann (사실상 상수).
Tarjan 1975 증명.

### 2.3 구현 (Python)
```python
class DSU:
    def __init__(self, n):
        self.parent = list(range(n))
        self.rank = [0] * n
    
    def find(self, x):
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])  # 경로 압축
        return self.parent[x]
    
    def union(self, a, b):
        ra, rb = self.find(a), self.find(b)
        if ra == rb: return False
        if self.rank[ra] < self.rank[rb]: ra, rb = rb, ra
        self.parent[rb] = ra
        if self.rank[ra] == self.rank[rb]: self.rank[ra] += 1
        return True
```

### 2.4 응용

- **Kruskal MST** — 사이클 감지
- **연결 요소 (Connected Components)**
- **Image segmentation**
- **Network connectivity (offline)**
- **Boggle / 단어 그리드 그룹화**

### 2.5 대표 문제

- LeetCode 547 Number of Provinces
- LeetCode 684 Redundant Connection
- LeetCode 200 Number of Islands (DFS 도 가능)
- LeetCode 990 Equations Possible

---

## 3. Segment Tree

### 3.1 개념

배열 A[0..N-1] 에 대해:
- 구간 합 / 최댓값 / 최솟값 / GCD 등 **결합법칙 (associative) 연산**.
- 점 갱신 O(log N), 구간 질의 O(log N).
- 구간 갱신 (Lazy Propagation) O(log N).

### 3.2 구조

```
A = [1, 3, 5, 7, 9, 11]
                          [36]
                       /        \
                   [9]            [27]
                  /    \          /    \
               [4]    [5]      [16]   [11]
              /  \           /     \
            [1] [3]        [7]    [9]
```

배열 표현: 1-indexed, 자식 = `2i`, `2i+1`. 크기 ≈ 4N.

### 3.3 구현 (Python — 구간 합)
```python
class SegTree:
    def __init__(self, n):
        self.n = n
        self.tree = [0] * (4 * n)
    
    def update(self, i, val, node=1, l=0, r=None):
        if r is None: r = self.n - 1
        if l == r:
            self.tree[node] = val
            return
        mid = (l + r) // 2
        if i <= mid: self.update(i, val, 2*node, l, mid)
        else:        self.update(i, val, 2*node+1, mid+1, r)
        self.tree[node] = self.tree[2*node] + self.tree[2*node+1]
    
    def query(self, ql, qr, node=1, l=0, r=None):
        if r is None: r = self.n - 1
        if qr < l or ql > r: return 0
        if ql <= l and r <= qr: return self.tree[node]
        mid = (l + r) // 2
        return self.query(ql, qr, 2*node, l, mid) + \
               self.query(ql, qr, 2*node+1, mid+1, r)
```

### 3.4 Lazy Propagation

구간 갱신을 O(log N) 으로. **lazy[]** 배열에 미적용 갱신 저장.

### 3.5 응용

- 동적 RMQ (Range Minimum Query)
- Range Sum + Point Update
- 좌표 압축 + Segment Tree
- 영속 (Persistent) Segment Tree — Kth 작은 수
- 2D Segment Tree

### 3.6 대표 문제

- LeetCode 307 Range Sum Query Mutable
- 백준 2042 구간 합 구하기
- 백준 1395 스위치
- 백준 2104 부분배열 고르기

---

## 4. Fenwick Tree (Binary Indexed Tree, BIT)

### 4.1 개념

Segment Tree 보다 **간단 & 메모리 절반**. Peter Fenwick 1994.
**누적합 (prefix sum)** 에 특화.

### 4.2 핵심 트릭

`i & -i` = i 의 LSB (Least Significant Bit). 이 비트만큼 인덱스 이동.

```
i = 12 = 1100₂
-i 의 LSB = 0100₂ = 4
i + LSB = 16 → 다음 갱신 위치
i - LSB = 8  → 다음 질의 위치
```

### 4.3 구현 (Python)
```python
class BIT:
    def __init__(self, n):
        self.n = n
        self.tree = [0] * (n + 1)
    
    def update(self, i, delta):
        i += 1
        while i <= self.n:
            self.tree[i] += delta
            i += i & -i
    
    def query(self, i):       # prefix sum [0..i]
        i += 1; s = 0
        while i > 0:
            s += self.tree[i]
            i -= i & -i
        return s
    
    def range_query(self, l, r):
        return self.query(r) - (self.query(l-1) if l > 0 else 0)
```

### 4.4 BIT vs Segment Tree

| 기준 | BIT | Segment Tree |
| --- | --- | --- |
| 코드 | 짧음 (10 줄) | 김 (30 줄+) |
| 메모리 | O(N) | O(4N) |
| 점 갱신 + 구간 합 | ✅ | ✅ |
| 구간 갱신 + 점 질의 | ✅ (변형) | ✅ |
| 구간 갱신 + 구간 합 | 어려움 | ✅ Lazy |
| RMQ | ❌ | ✅ |
| 2D | ✅ 간단 | ✅ 복잡 |

### 4.5 응용

- **Inversion count** — 정렬 시 swap 수
- 부분합 / 누적 카운트
- 좌표 압축 + BIT
- 2D BIT — 행렬 부분합

### 4.6 대표 문제

- LeetCode 315 Count of Smaller Numbers After Self
- 백준 7795 먹을 것인가 먹힐 것인가
- 백준 1517 버블 소트

---

## 5. Bloom Filter

### 5.1 개념

**확률적** 집합 자료구조. 멤버십 질문 ("x 가 집합에 있나?"):
- "있을 수도 있다" — 가짜 양성 (false positive) 가능
- "확실히 없다" — 가짜 음성 (false negative) **불가능**

Burton Howard Bloom 1970.

### 5.2 동작

- 크기 m 의 비트 배열 + k 개 해시 함수.
- 삽입: x 의 k 해시 → m 비트 중 k 곳 set.
- 질의: k 곳 모두 set 이면 "있을 수도", 하나라도 unset 이면 "없음".

### 5.3 가짜 양성률

n 개 삽입, m 비트, k 해시:
```
FPR ≈ (1 - e^(-kn/m))^k
최적 k = (m/n) * ln(2)
```

n=1M, FPR=1% 원하면 m ≈ 9.6 N → 약 9.6 비트 / 원소.

### 5.4 구현 (Python)
```python
import hashlib, math

class BloomFilter:
    def __init__(self, n, fpr=0.01):
        self.m = int(-n * math.log(fpr) / (math.log(2)**2))
        self.k = int(self.m / n * math.log(2))
        self.bits = bytearray(self.m // 8 + 1)
    
    def _hashes(self, x):
        s = str(x).encode()
        for i in range(self.k):
            h = int(hashlib.sha256(s + bytes([i])).hexdigest(), 16) % self.m
            yield h
    
    def add(self, x):
        for h in self._hashes(x):
            self.bits[h // 8] |= 1 << (h % 8)
    
    def __contains__(self, x):
        return all(self.bits[h // 8] & (1 << (h % 8)) for h in self._hashes(x))
```

### 5.5 응용

- **Chrome Safe Browsing** — 위험 URL 첫 단계 필터
- **Cassandra / HBase / Bigtable** — SSTable 룩업 회피
- **CDN 캐시** — 첫 조회 차단
- **분산 DB / NoSQL** — Bloom Joins
- **이메일 스팸** — 알려진 스패머 IP
- **암호화폐 (Bitcoin SPV)** — 라이트 클라이언트

### 5.6 변형

- **Counting Bloom Filter** — 삭제 가능 (비트 → 카운터)
- **Cuckoo Filter** — 더 나은 공간 효율 + 삭제
- **Scalable Bloom Filter** — 동적 크기

### 5.7 함정

- 가짜 양성률 사전 계획 필수
- 너무 가득 차면 FPR 폭발
- 해시 함수 독립성 필요 — Double Hashing 트릭

---

## 6. Skip List

### 6.1 개념

**확률적 균형** 자료구조. William Pugh 1989.
- 정렬 연결 리스트 + 여러 레벨의 "express lane".
- 평균 O(log N) — Red-Black Tree 와 동일.
- 구현 훨씬 단순.

### 6.2 구조

```
Level 3: 1 ──────────────→ 9 ──────→ NIL
Level 2: 1 ──→ 4 ────────→ 9 ──────→ NIL
Level 1: 1 ──→ 4 ──→ 7 ──→ 9 ──→ 12 → NIL
Level 0: 1 → 3 → 4 → 6 → 7 → 9 → 12 → NIL
```

각 노드는 확률 p (보통 1/2) 로 위 레벨에 promote.

### 6.3 구현 (Python — 간략)
```python
import random

class SkipNode:
    def __init__(self, key, level):
        self.key = key
        self.forward = [None] * (level + 1)

class SkipList:
    MAX_LEVEL = 16
    P = 0.5
    
    def __init__(self):
        self.head = SkipNode(-float('inf'), self.MAX_LEVEL)
        self.level = 0
    
    def _random_level(self):
        lvl = 0
        while random.random() < self.P and lvl < self.MAX_LEVEL:
            lvl += 1
        return lvl
    
    def insert(self, key):
        update = [None] * (self.MAX_LEVEL + 1)
        x = self.head
        for i in range(self.level, -1, -1):
            while x.forward[i] and x.forward[i].key < key:
                x = x.forward[i]
            update[i] = x
        lvl = self._random_level()
        if lvl > self.level:
            for i in range(self.level + 1, lvl + 1):
                update[i] = self.head
            self.level = lvl
        new = SkipNode(key, lvl)
        for i in range(lvl + 1):
            new.forward[i] = update[i].forward[i]
            update[i].forward[i] = new
    
    def search(self, key):
        x = self.head
        for i in range(self.level, -1, -1):
            while x.forward[i] and x.forward[i].key < key:
                x = x.forward[i]
        x = x.forward[0]
        return x is not None and x.key == key
```

### 6.4 응용

- **Redis Sorted Set (ZSET)** — Skip List + Hash Map.
- **LevelDB / RocksDB MemTable** — Skip List 기반.
- **동시성 환경** — Lock-free Skip List 가 트리보다 쉬움.
- **인메모리 인덱스**

### 6.5 Skip List vs Balanced BST

| 기준 | Skip List | Red-Black Tree |
| --- | --- | --- |
| 평균 O() | O(log N) | O(log N) |
| 최악 | O(N) — 확률적 | O(log N) |
| 구현 | 단순 | 복잡 |
| 동시성 | 쉬움 | 어려움 |
| 메모리 | 약 2N 포인터 | 2N + color |
| 캐시 효율 | 비슷 | 비슷 |

---

## 7. 기타 고급 자료구조 (인덱스)

| 구조 | 용도 | 참고 |
| --- | --- | --- |
| **Sparse Table** | 정적 RMQ O(1) 질의 | 전처리 O(N log N) |
| **Heavy-Light Decomposition** | 트리 경로 질의 | 경쟁 프로그래밍 |
| **Link/Cut Tree** | 동적 트리 | Sleator-Tarjan |
| **Wavelet Tree** | 부분배열 Kth | 정보 검색 |
| **Suffix Automaton** | 문자열 부분문자열 카운트 | O(N) 구축 |
| **Persistent Data Structure** | 이전 버전 보존 | 함수형 / Kth |
| **kd-Tree / R-Tree** | 공간 인덱싱 | GIS / DB |
| **Treap** | 확률적 BST | 우선순위 + 키 |
| **Splay Tree** | 자가 조정 BST | 캐시 친화 |
| **Rope** | 거대 문자열 편집 | 텍스트 에디터 |

---

## 8. 함정 / 안티패턴

### 함정 1 — 과도한 자료구조
N=100 문제에 Segment Tree 필요 없음. **데이터 크기로 결정**.

### 함정 2 — Bloom Filter FPR
사전 설계 없이 막 쓰면 50% FPR.

### 함정 3 — DSU 의 union by rank 빠뜨림
경로 압축만 있고 union by rank 없으면 O(log N) 만 보장.

### 함정 4 — Segment Tree 의 배열 크기
4N 이 안전. 2N 으로는 부족할 수 있음.

### 함정 5 — Skip List 의 최악 케이스
악의적 입력 가능 → 시드 비밀화.

---

## 9. 학습 자료

- CLRS Ch.21 (Disjoint Set), Ch.14 (Augmented DS)
- **Competitive Programming Handbook** (Antti Laaksonen) — Ch.9-10
- [VisuAlgo — Segment Tree / Union-Find](https://visualgo.net/en)
- [cp-algorithms.com](https://cp-algorithms.com/) — Segment Tree, BIT 깊이 있게
- William Pugh 1989 — Skip List 원논문

---

## 10. 대표 문제 모음

| 문제 | 영역 | 자료구조 |
| --- | --- | --- |
| LeetCode 547 Number of Provinces | DSU | Union-Find |
| LeetCode 1971 Find Path in Graph | DSU | Union-Find |
| LeetCode 307 Range Sum Query Mutable | 구간 합 | BIT or SegTree |
| LeetCode 315 Count Smaller After Self | 역순 | BIT |
| 백준 2042 구간 합 | 구간 합 + 갱신 | SegTree |
| 백준 11505 구간 곱 | 구간 곱 | SegTree |
| LeetCode 705 Design HashSet | 집합 | (Bloom 확장 가능) |
| LeetCode 1206 Design Skiplist | Skip List | Skip List |

---

## 11. 관련

- [[../trees/trees]] — Balanced BST 와 Skip List 비교
- [[../hash-tables/hash-tables]] — Bloom Filter 확장
- [[../graphs/graphs]] — DSU 의 MST 응용
- [[../../algorithm/graph-theory/graph-theory]] — Kruskal
- [[../../algorithm/algorithm|↑ algorithm]]
- [[../data-structure|↑ data-structure]]
