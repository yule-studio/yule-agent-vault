---
title: "Priority Queue — 우선순위 큐"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T15:05:00+09:00
tags:
  - data-structure
  - priority-queue
---

# Priority Queue — 우선순위 큐

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | API / 구현 비교 |

**[[heaps|↑ Heaps]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

**우선순위가 가장 높은 (또는 낮은) 원소** 가 먼저 나오는 큐. Heap / 정렬 array / 정렬 list 로 구현.

---

## 2. API

| | 설명 | 시간 (heap) |
| --- | --- | --- |
| `push(item, priority)` | 추가 | O(log N) |
| `pop()` | 최고 우선순위 제거 + 반환 | O(log N) |
| `peek()` | 최고 우선순위 보기 | O(1) |
| `is_empty()` | 비어있나 | O(1) |

---

## 3. 구현 비교

| 구현 | Push | Pop | Peek |
| --- | --- | --- | --- |
| Sorted array | O(N) | O(1) | O(1) |
| Unsorted array | O(1) | O(N) | O(N) |
| Sorted linked list | O(N) | O(1) | O(1) |
| Unsorted linked list | O(1) | O(N) | O(N) |
| **Binary Heap** | O(log N) | O(log N) | O(1) |
| **Binomial Heap** | O(log N) | O(log N) | O(1) |
| **Fibonacci Heap** | O(1) | O(log N) | O(1) |
| BST (balanced) | O(log N) | O(log N) | O(log N) (min/max) |

### 결론 — Binary Heap 가 보통 best
- 균형 잡힘
- Cache 친화 (array)
- 구현 단순

---

## 4. Python 예 — `heapq` wrapping

```python
import heapq

class PriorityQueue:
    def __init__(self):
        self.heap = []
        self.counter = 0    # tiebreaker
    
    def push(self, priority, item):
        heapq.heappush(self.heap, (priority, self.counter, item))
        self.counter += 1
    
    def pop(self):
        return heapq.heappop(self.heap)[2]
    
    def peek(self):
        return self.heap[0][2] if self.heap else None
    
    def is_empty(self):
        return not self.heap
```

### Counter 의 의미
- Priority 같으면 — item 비교 (item 이 not comparable 가능)
- Counter — insertion order tiebreaker

---

## 5. Java `PriorityQueue`

```java
PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);
pq.offer(new int[]{5, 100});
pq.offer(new int[]{1, 50});
pq.offer(new int[]{3, 75});
pq.poll();    // [1, 50] (lowest priority)
```

### Properties
- Default min-heap
- Custom comparator
- Iterator order — undefined (not sorted)

---

## 6. C++ `priority_queue`

```cpp
#include <queue>

// Default — max-heap
std::priority_queue<int> max_pq;

// Min-heap
std::priority_queue<int, std::vector<int>, std::greater<>> min_pq;

// Custom
auto cmp = [](const Task& a, const Task& b) { return a.priority > b.priority; };
std::priority_queue<Task, std::vector<Task>, decltype(cmp)> pq(cmp);
```

---

## 7. 응용

### 7.1 Dijkstra Shortest Path
```python
def dijkstra(graph, start):
    dist = {n: float('inf') for n in graph}
    dist[start] = 0
    pq = [(0, start)]
    while pq:
        d, u = heapq.heappop(pq)
        if d > dist[u]: continue        # stale
        for v, w in graph[u]:
            new_d = d + w
            if new_d < dist[v]:
                dist[v] = new_d
                heapq.heappush(pq, (new_d, v))
    return dist
```

### 7.2 A* Search
- Priority = f(n) = g(n) + h(n)
- Pathfinding (games)

### 7.3 Top-K elements
```python
def top_k(arr, k):
    heap = []
    for x in arr:
        heapq.heappush(heap, x)
        if len(heap) > k:
            heapq.heappop(heap)
    return sorted(heap, reverse=True)
```

→ O(N log k) — 전체 정렬보다 빠름.

### 7.4 K-th largest
- 같은 — 작은 heap N → top-k 반환

### 7.5 Median maintenance
- Two heaps (min + max)
- 자세히 → [[heap-applications]]

### 7.6 Merge K sorted lists
```python
def merge_k(lists):
    pq = [(lst[0], i, 0) for i, lst in enumerate(lists) if lst]
    heapq.heapify(pq)
    result = []
    while pq:
        val, list_idx, elem_idx = heapq.heappop(pq)
        result.append(val)
        if elem_idx + 1 < len(lists[list_idx]):
            next_val = lists[list_idx][elem_idx + 1]
            heapq.heappush(pq, (next_val, list_idx, elem_idx + 1))
    return result
```

### 7.7 Task scheduler — OS
- Process 의 priority
- I/O 처리 순서

### 7.8 Huffman coding
- 가장 적은 빈도의 char 2 개 결합
- Heap 으로 매번 minimum 2

---

## 8. Indexed Priority Queue

### 정의
- Item → priority 변경 가능 (decrease-key)

### 사용
- Dijkstra — 노드 별 distance 갱신
- Prim's MST

### Python
- heapq — 직접 X (lazy deletion 흔함)
- `pqdict` 라이브러리

### Java
- `PriorityQueue.remove(o)` — O(N), 비싸
- 자체 — indexed heap 구현

---

## 9. Lazy deletion 패턴

### 정의
- Decrease-key 대신 — 옛 entry 무시

```python
def dijkstra_lazy(graph, start):
    dist = {n: float('inf') for n in graph}
    dist[start] = 0
    pq = [(0, start)]
    while pq:
        d, u = heapq.heappop(pq)
        if d > dist[u]:                # lazy delete — stale entry skip
            continue
        for v, w in graph[u]:
            new_d = d + w
            if new_d < dist[v]:
                dist[v] = new_d
                heapq.heappush(pq, (new_d, v))
```

### 효과
- 코드 단순
- Heap 크기 ↑ (stale entries)

### 단점
- 최악 — 모든 edge — heap 에 — N×E 크기
- 실제 — 거의 문제 없음

---

## 10. Bounded / Capacity Limited

### 정의
- 크기 한계 (k)
- Top-k 만 유지

```python
class BoundedHeap:
    def __init__(self, k):
        self.k = k
        self.heap = []
    
    def add(self, x):
        if len(self.heap) < self.k:
            heapq.heappush(self.heap, x)
        elif x > self.heap[0]:
            heapq.heappushpop(self.heap, x)
```

---

## 11. Concurrent Priority Queue

### 도전
- 락 contention
- 분산 — 일관성

### 도구
- Java `PriorityBlockingQueue`
- Lock-free PQ (Sundell, 2003)

---

## 12. 함정

### 함정 1 — Python heapq tuple 비교
같은 priority — 다음 element 비교. Item not comparable → error.
Counter / id 추가.

### 함정 2 — Java `remove(o)` 의 O(N)
arbitrary 삭제 — 비싸. lazy deletion.

### 함정 3 — Iteration 의 순서
PQ 의 iteration — undefined order. Pop 으로 정렬 추출.

### 함정 4 — Max heap 의 부호 반전
부호 반전 — overflow (큰 음수). 항상 negate 후 negate.

### 함정 5 — Stable 아님
같은 priority — FIFO 보장 X. Counter 로 stable.

### 함정 6 — Decrease-key 누락
없는 heap — re-push (with stale check).

---

## 13. 학습 자료

- CLRS Ch 6
- Algorithm Design Manual — Skiena Ch 4
- "Algorithms" Sedgewick — Priority Queue

---

## 14. 관련

- [[heaps]] — Hub
- [[binary-heap]] — 기반
- [[heap-applications]]
- [[../advanced/fibonacci-heap]] (TBD)
