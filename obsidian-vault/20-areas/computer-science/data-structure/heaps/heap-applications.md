---
title: "Heap 응용 — Top-K / Median / Dijkstra"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T15:10:00+09:00
tags:
  - data-structure
  - heap
  - applications
---

# Heap 응용

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | top-K / median / merge / Dijkstra |

**[[heaps|↑ Heaps]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 응용 — 한눈

| 응용 | 자료구조 | 시간 |
| --- | --- | --- |
| Top-K largest | Min-heap of size k | O(N log k) |
| K-th largest | Min-heap of size k | O(N log k) |
| Running median | 2 heaps (min + max) | O(log N) per |
| Merge K sorted | Min-heap | O(N log k) |
| Dijkstra | Min-heap | O((V+E) log V) |
| Heap sort | Max-heap | O(N log N) |
| Huffman coding | Min-heap | O(N log N) |
| Top-K frequent | Min-heap | O(N log k) |

---

## 2. Top-K Largest

### 문제
- 큰 배열 N — 상위 k 개

### Naive
- 정렬 — O(N log N)

### Heap
```python
import heapq

def top_k(arr, k):
    heap = []
    for x in arr:
        if len(heap) < k:
            heapq.heappush(heap, x)
        elif x > heap[0]:
            heapq.heappushpop(heap, x)
    return sorted(heap, reverse=True)
```

### 시간 — O(N log k)
- k << N — 정렬보다 빠름
- k = N — 같음

### Streaming
- 큰 streaming data — heap 만 유지 (전체 저장 X)
- O(k) 메모리

---

## 3. K-th Largest

```python
def find_kth_largest(arr, k):
    heap = []
    for x in arr:
        heapq.heappush(heap, x)
        if len(heap) > k:
            heapq.heappop(heap)
    return heap[0]    # k 번째 largest
```

### Quickselect — O(N) average
- 정렬 안 함 — partition 만
- Worst — O(N²), introselect 가 O(N) 보장

---

## 4. Running Median (Two Heaps)

### 문제
- Stream 의 매 시점 — 중앙값

### 해결
- **Max-heap (low half)** — 작은 절반
- **Min-heap (high half)** — 큰 절반
- 두 heap 의 크기 차이 ≤ 1

```python
class MedianFinder:
    def __init__(self):
        self.low = []      # max-heap (negated)
        self.high = []     # min-heap
    
    def add_num(self, num):
        # Push to low first
        heapq.heappush(self.low, -num)
        # Move max of low to high
        heapq.heappush(self.high, -heapq.heappop(self.low))
        # Balance
        if len(self.high) > len(self.low):
            heapq.heappush(self.low, -heapq.heappop(self.high))
    
    def find_median(self):
        if len(self.low) > len(self.high):
            return -self.low[0]
        return (-self.low[0] + self.high[0]) / 2
```

### 시간 — O(log N) per add, O(1) median

---

## 5. Merge K Sorted Lists

### 문제
- N 개의 정렬 list — 하나로 merge

### Heap
```python
def merge_k_sorted(lists):
    pq = []
    for i, lst in enumerate(lists):
        if lst:
            heapq.heappush(pq, (lst[0], i, 0))
    
    result = []
    while pq:
        val, list_idx, elem_idx = heapq.heappop(pq)
        result.append(val)
        if elem_idx + 1 < len(lists[list_idx]):
            next_val = lists[list_idx][elem_idx + 1]
            heapq.heappush(pq, (next_val, list_idx, elem_idx + 1))
    return result
```

### 시간 — O(N log k) (N = 전체 원소, k = list 수)

### 응용
- External sort (file)
- Distributed log merge
- LSM-tree compaction

---

## 6. Dijkstra Shortest Path

```python
def dijkstra(graph, start):
    dist = {n: float('inf') for n in graph}
    dist[start] = 0
    pq = [(0, start)]
    while pq:
        d, u = heapq.heappop(pq)
        if d > dist[u]: continue
        for v, w in graph[u]:
            new_d = d + w
            if new_d < dist[v]:
                dist[v] = new_d
                heapq.heappush(pq, (new_d, v))
    return dist
```

### 시간 — O((V+E) log V) (binary heap)
- Fibonacci heap — O(E + V log V)
- 실제 — binary heap 더 빠름 (constant factor)

자세히 → [[../graphs/shortest-path]] (TBD)

---

## 7. Prim's MST

### 비슷한 Dijkstra
- 가장 작은 edge 가져가기
- 최소 신장 트리

자세히 → [[../graphs/mst]] (TBD)

---

## 8. Heap Sort

```python
def heap_sort(arr):
    # 1. Build max-heap O(N)
    n = len(arr)
    for i in range(n // 2 - 1, -1, -1):
        sift_down(arr, i, n)
    
    # 2. Extract O(N log N)
    for end in range(n - 1, 0, -1):
        arr[0], arr[end] = arr[end], arr[0]
        sift_down(arr, 0, end)
    return arr
```

### 시간 — O(N log N) (worst-case)
### 공간 — O(1) in-place
### 단점 — Cache 부담 (Quicksort 보다 느림)

---

## 9. Huffman Coding

### 정의
- 변동 길이 인코딩 — 자주 나오는 char 짧게

### 알고리즘
```python
def huffman(freqs):
    heap = [(f, c) for c, f in freqs.items()]
    heapq.heapify(heap)
    while len(heap) > 1:
        f1, n1 = heapq.heappop(heap)
        f2, n2 = heapq.heappop(heap)
        heapq.heappush(heap, (f1 + f2, [n1, n2]))
    return heap[0][1]
```

### 사용
- ZIP / DEFLATE
- JPEG (after DCT)

---

## 10. Top-K Frequent Elements

```python
def top_k_frequent(nums, k):
    count = Counter(nums)
    return heapq.nlargest(k, count.keys(), key=count.get)
```

### 또는 Bucket sort — O(N)

---

## 11. Sliding Window Maximum

### Heap 대신 — Monotonic deque (O(N))
- Heap — O(N log k) (lazy deletion 필요)
- Deque — 더 빠름

자세히 → [[../stacks-and-queues/monotonic-stack-deque]]

---

## 12. Task Scheduler

### CPU / OS — Process priority
- Linux CFS (Completely Fair Scheduler) — Red-Black Tree
- Priority queue 변형

### Job queue (real-time)
- Earliest deadline first (EDF)
- Heap by deadline

---

## 13. Memory allocation

### Best-fit / Worst-fit
- Free blocks — priority queue
- Best — smallest sufficient block (min-heap by size)

### Slab allocator (Linux)
- Multiple heaps by size class

---

## 14. A* Search

### Priority = f(n) = g(n) + h(n)
- g — actual cost
- h — heuristic

### 사용
- Pathfinding (게임)
- Navigation (Google Maps)

---

## 15. Network packet scheduling

### QoS — Priority queue
- 음성 (높음) > 비디오 > 데이터 (낮음)
- Router 가 우선순위

### Weighted Fair Queueing
- 각 flow 별 차례
- 더 공평

자세히 → [[../../network/topics/bufferbloat]]

---

## 16. 함정

### 함정 1 — k > N
top-k of N (k > N) — 전체 반환. 검사.

### 함정 2 — Stable order
같은 priority — FIFO X. Counter 로 tiebreaker.

### 함정 3 — Median 의 두 heap balance
크기 차이 ≤ 1 — 항상 보장. 한쪽 너무 크면 옮김.

### 함정 4 — Dijkstra 의 stale entries
Decrease-key 대신 — lazy delete (`if d > dist[u]: continue`).

### 함정 5 — Heap 의 사용자 큰 객체
큰 객체 — swap 비용 ↑. Index / id 만 heap.

### 함정 6 — Heap 의 update X
배열 변경 — heap property 깨짐. 새 entry push.

---

## 17. 학습 자료

- CLRS Ch 6 — Heap Sort, Priority Queue
- "Algorithm Design Manual" — Heap
- LeetCode Heap tag

---

## 18. 관련

- [[heaps]] — Hub
- [[binary-heap]] / [[priority-queue]]
- [[../graphs/shortest-path]] (TBD)
- [[../../algorithm/algorithm]] — Dijkstra / Sort
