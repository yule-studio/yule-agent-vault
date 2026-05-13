---
title: "Binary Heap — 이진 힙"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T15:00:00+09:00
tags:
  - data-structure
  - heap
---

# Binary Heap — 이진 힙

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Array 표현 / heapify |

**[[heaps|↑ Heaps]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

**완전 이진 트리 + heap property** (parent ≤ children for min-heap). Array 로 표현. Priority queue 의 기반.

---

## 2. 구조

```
       1
     /   \
    3     5
   / \   / \
  4   7 9   8

Array: [1, 3, 5, 4, 7, 9, 8]
       index 0 부터
```

### Index 관계 (0-based)
- parent(i) = (i - 1) / 2
- left(i)   = 2i + 1
- right(i)  = 2i + 2

### Heap property
- **Min-heap** — parent ≤ children
- **Max-heap** — parent ≥ children

---

## 3. 연산

| | 시간 |
| --- | --- |
| `peek` (root) | O(1) |
| `push` | O(log N) |
| `pop` (root) | O(log N) |
| `heapify` (build) | O(N) |
| `decrease-key` | O(log N) |
| `delete` (arbitrary) | O(log N) (위치 알면) |

---

## 4. Push (sift up)

```python
def push(heap, val):
    heap.append(val)
    i = len(heap) - 1
    while i > 0:
        parent = (i - 1) // 2
        if heap[parent] > heap[i]:    # min-heap
            heap[parent], heap[i] = heap[i], heap[parent]
            i = parent
        else:
            break
```

### 흐름
1. 마지막에 추가
2. parent 보다 작으면 swap
3. heap property 회복까지

---

## 5. Pop (sift down)

```python
def pop(heap):
    if not heap: return None
    root = heap[0]
    last = heap.pop()
    if heap:
        heap[0] = last
        i = 0
        n = len(heap)
        while True:
            l, r = 2*i + 1, 2*i + 2
            smallest = i
            if l < n and heap[l] < heap[smallest]:
                smallest = l
            if r < n and heap[r] < heap[smallest]:
                smallest = r
            if smallest == i: break
            heap[i], heap[smallest] = heap[smallest], heap[i]
            i = smallest
    return root
```

### 흐름
1. root 저장 (반환용)
2. 마지막 원소 → root
3. 작은 자식과 swap (heap property 회복)

---

## 6. Heapify — Build O(N)

### Naive — O(N log N)
```python
def build_heap(arr):
    heap = []
    for x in arr:
        push(heap, x)
    return heap
```

### Optimal — O(N)
```python
def build_heap(arr):
    n = len(arr)
    for i in range(n // 2 - 1, -1, -1):
        sift_down(arr, i, n)
    return arr
```

### 왜 O(N)?
- 마지막 레벨 (N/2 노드) — sift down 0 단계
- 그 위 레벨 (N/4 노드) — 1 단계
- ...
- 총 작업 = Σ (h × N/2^(h+1)) = O(N)

---

## 7. Python `heapq`

```python
import heapq

heap = [3, 1, 4, 1, 5, 9, 2, 6]
heapq.heapify(heap)         # O(N)
# heap == [1, 1, 2, 3, 5, 9, 4, 6]

heapq.heappush(heap, 0)     # O(log N)
heapq.heappop(heap)          # O(log N) — min

# heapq — min heap only
# Max heap 흉내 — 부호 반전
heapq.heappush(heap, -x)
-heapq.heappop(heap)

# nlargest / nsmallest
heapq.nlargest(3, [1, 5, 2, 8, 3])    # [8, 5, 3]
heapq.nsmallest(3, [1, 5, 2, 8, 3])   # [1, 2, 3]
```

---

## 8. Java `PriorityQueue`

```java
PriorityQueue<Integer> pq = new PriorityQueue<>();    // min-heap
pq.offer(3);
pq.offer(1);
pq.offer(4);
pq.poll();    // 1
pq.peek();    // 3

// Max-heap
PriorityQueue<Integer> max = new PriorityQueue<>(Comparator.reverseOrder());

// Custom comparator
PriorityQueue<int[]> pq2 = new PriorityQueue<>((a, b) -> a[1] - b[1]);
```

---

## 9. C++

```cpp
#include <queue>
std::priority_queue<int> max_heap;    // max-heap (default)
std::priority_queue<int, std::vector<int>, std::greater<>> min_heap;

max_heap.push(3);
max_heap.top();      // 3
max_heap.pop();
```

---

## 10. Heapsort

```python
def heap_sort(arr):
    # 1. Build max-heap
    n = len(arr)
    for i in range(n // 2 - 1, -1, -1):
        sift_down(arr, i, n)
    
    # 2. Extract one by one
    for end in range(n - 1, 0, -1):
        arr[0], arr[end] = arr[end], arr[0]
        sift_down(arr, 0, end)
    return arr
```

### 시간 — O(N log N)
### 공간 — O(1) (in-place)
### 단점 — Quicksort 보다 느림 (cache locality)

---

## 11. Heap vs BST

| | Binary Heap | BST |
| --- | --- | --- |
| **Min/Max** | O(1) | O(log N) |
| **Insert** | O(log N) | O(log N) |
| **Delete root** | O(log N) | O(log N) |
| **Search** | O(N) | O(log N) |
| **Sorted iteration** | O(N log N) (pop) | O(N) |
| **메모리** | Array (compact) | 노드 (포인터 overhead) |

### 선택
- 최댓값 / 최솟값만 — Heap
- 정렬 / 범위 — BST

---

## 12. Indexed / Decrease-key

### 문제
- 임의 원소 — 우선순위 변경 (Dijkstra)

### 해결
- Position map (hash) — value → index
- Decrease-key — index 알면 sift up O(log N)

```python
class IndexedHeap:
    def __init__(self):
        self.heap = []
        self.pos = {}    # value → index
    
    def decrease_key(self, val, new_priority):
        i = self.pos[val]
        self.heap[i] = (new_priority, val)
        # sift up
```

---

## 13. d-ary Heap

### 정의
- d-ary (d=2 — binary, d=4 — quaternary)
- d=4 — push 빠름 (트리 얕음), pop 느림 (자식 더 많음)
- 실험적 — d=4 좋음

자세히 → algorithm-advanced

---

## 14. 함정

### 함정 1 — Index 계산 — 0 vs 1 base
0-base: parent = (i-1)/2, child = 2i+1, 2i+2
1-base: parent = i/2, child = 2i, 2i+1

### 함정 2 — Python heapq 는 min-only
Max — `-x` 사용 (큰 수 -∞).

### 함정 3 — Tuple 비교
heappush(heap, (priority, item)) — priority 같으면 item 비교.
Item 비교 불가 — TypeError. counter 사용:
```python
heapq.heappush(heap, (priority, counter, item))
counter += 1
```

### 함정 4 — Lazy deletion
Decrease-key 대신 — 옛 entry 무시 (pop 시 skip).
빈번 — heap 크기 폭증.

### 함정 5 — Heap 의 sorted X
Array 자체 — 정렬 X. Iteration 순서 의미 X.

### 함정 6 — Heapify 의 O(N) 만 사용
N push 는 O(N log N). 한꺼번에 — heapify.

### 함정 7 — Custom comparator 의 equals
대부분 — strict comparison. Equality 처리 별도.

---

## 15. 학습 자료

- CLRS Ch 6
- Python `heapq` docs
- "Algorithms" Sedgewick — Priority Queue

---

## 16. 관련

- [[heaps]] — Hub
- [[priority-queue]] — Wrapping
- [[heap-applications]] — Dijkstra / top-K / median
- [[../advanced/fibonacci-heap]] (TBD)
