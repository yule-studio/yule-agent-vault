---
title: "힙 (Heaps) / 우선순위 큐"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T08:00:00+09:00
tags:
  - data-structure
  - heap
  - priority-queue
---

# 힙 (Heaps) / 우선순위 큐 (Priority Queue)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 수준 + 11 언어 |

**[[../data-structure|↑ data-structure]]** · **[[../../computer-science|↑↑ computer-science]]**

---

## 1. 한 줄 정의

**완전 이진 트리 (Complete Binary Tree)** 인데 **부모-자식 간 순서 조건**을
가진 자료구조. 최솟값 (min-heap) 또는 최댓값 (max-heap) 을 **O(log N)** 에
삽입/삭제. 우선순위 큐의 표준 구현.

---

## 2. 개념의 깊이

### 2.1 왜 힙인가

"매번 가장 작은 (또는 큰) 원소를 빠르게 빼내야 한다" 가 자주 등장:
- Dijkstra 최단 경로 — 미방문 중 거리 최소 노드
- 작업 스케줄러 — 우선순위 최고 작업
- Huffman 코딩 — 빈도 최소 두 노드 병합
- 이벤트 시뮬레이션 — 가장 빠른 이벤트
- top-K — K개 최대 / 최소

배열·BST 로 도 가능하지만:
- 정렬된 배열: 삽입 O(N), top O(1) — 삽입 너무 느림
- BST: 모든 연산 O(log N) — top 도 O(log N)
- **Heap**: 삽입 O(log N), top O(1), pop O(log N) — 최적

### 2.2 Heap Property (힙 성질)

- **Min-heap**: 모든 노드 ≤ 자식들 (루트가 최솟값).
- **Max-heap**: 모든 노드 ≥ 자식들 (루트가 최댓값).

**부분 순서** — 형제 노드 간 순서 없음. 정렬된 자료구조 X.

### 2.3 배열 표현 (Implicit Tree)

완전 이진 트리 → 배열에 **포인터 없이** 저장:

```
인덱스: 0  1  2  3  4  5  6  7  8  9
값:    [-, 1, 3, 5, 7, 9, 6, 10, 14, 12]
        ↑(unused or 0-indexed)

부모(i) = (i - 1) / 2   (1-indexed: i / 2)
왼쪽(i) = 2 * i + 1     (1-indexed: 2*i)
오른쪽(i) = 2 * i + 2   (1-indexed: 2*i + 1)
```

트리 그림:
```
            1
          /   \
         3     5
        / \   / \
       7   9 6  10
      / \
    14  12
```

배열 인덱스로 부모/자식 즉시 계산 → **포인터 X, 메모리 효율, 캐시 친화**.

### 2.4 핵심 연산 — Sift up / Sift down

#### Sift up (Bubble up) — 삽입 시
```
새 원소를 마지막에 추가 → 부모와 비교 → 부모보다 작으면 swap → 반복

INSERT(x):
    a.append(x)
    i = len(a) - 1
    while i > 0 and a[i] < a[(i-1)/2]:
        swap(a[i], a[(i-1)/2])
        i = (i-1)/2
```

#### Sift down (Heapify) — 삭제 시
```
루트를 마지막 원소로 교체 → 작은 자식과 비교 → 자식보다 크면 swap → 반복

EXTRACT_MIN():
    min_val = a[0]
    a[0] = a.pop()
    i = 0
    while 2*i + 1 < len(a):
        smallest = i
        if a[2*i+1] < a[smallest]: smallest = 2*i+1
        if 2*i+2 < len(a) and a[2*i+2] < a[smallest]: smallest = 2*i+2
        if smallest == i: break
        swap(a[i], a[smallest])
        i = smallest
    return min_val
```

### 2.5 Build Heap — O(N)?!

N 개 원소로 힙 만들기:
- 단순 방법: N 번 insert → O(N log N).
- **Heapify (Floyd, 1964)**: 마지막 부모부터 sift down → **O(N)**!

```python
def heapify(arr):
    n = len(arr)
    for i in range(n // 2 - 1, -1, -1):   # 마지막 부모부터
        sift_down(arr, i, n)
```

**왜 O(N)?**

각 노드의 sift down 비용은 **그 노드의 높이** 에 비례. 트리의 노드 대부분 (절반) 은
잎 → 높이 0. 분석:

```
잎 노드 N/2 개: 비용 0
높이 1 노드 N/4 개: 비용 1
높이 2 노드 N/8 개: 비용 2
...
높이 h = log N 노드 1 개: 비용 log N

총 비용 = Σ (N / 2^(k+1)) × k  (k = 1 ~ log N)
       = N × Σ k / 2^(k+1)
       = N × O(1)   (geometric series)
       = O(N)
```

### 2.6 Heap Sort — O(N log N)

1. Build heap O(N).
2. 루트 (max) ↔ 마지막 원소 swap → 힙 크기 -1.
3. Sift down 으로 복원 O(log N).
4. N 번 반복 → O(N log N).

**장점**: 추가 메모리 O(1) (in-place).
**단점**: 캐시 비친화 (배열 멀리 떨어진 원소 비교) → 실제로 quick sort 보다 느림.

### 2.7 우선순위 큐 ADT vs 힙

- **우선순위 큐**: ADT (인터페이스).
- **힙**: 구현 중 하나 (가장 흔함).

다른 구현:
- 정렬된 배열 / Linked List (느림).
- BST (Red-Black) — 임의 원소 삭제 O(log N).
- Fibonacci Heap — decrease-key O(1) amortized.
- Pairing Heap — Fibonacci 의 단순화.

### 2.8 역사

- **1964 J.W.J. Williams** — Heap sort 의 발명 + 힙 자료구조.
- **1964 Robert W. Floyd** — Build heap 의 O(N) 알고리즘.
- **1980 Vuillemin** — Binomial Heap.
- **1987 Fredman & Tarjan** — Fibonacci Heap (Dijkstra 의 이론적 최적화).

---

## 3. 그림 — Sift Up 단계별

힙: `[1, 3, 5, 7, 9, 6, 10]`. 새 원소 2 삽입.

```
Step 1: 끝에 추가
              1
            /   \
           3     5
          / \   / \
         7   9 6  10
        /
       2
       ↑

Step 2: 부모 7 과 비교. 2 < 7 → swap.
              1
            /   \
           3     5
          / \   / \
         2   9 6  10
        /
       7

Step 3: 부모 3 과 비교. 2 < 3 → swap.
              1
            /   \
           2     5
          / \   / \
         3   9 6  10
        /
       7

Step 4: 부모 1 과 비교. 2 > 1 → 종료.
최종: [1, 2, 5, 3, 9, 6, 10, 7]
```

---

## 4. 복잡도

| 연산 | 시간 |
| --- | --- |
| Insert | O(log N) |
| Extract min/max | O(log N) |
| Peek (top) | O(1) |
| Build heap (N 원소) | **O(N)** |
| Decrease-key (위치 알 때) | O(log N) |
| 임의 원소 삭제 | O(N) (위치 찾기) → 위치 알면 O(log N) |
| Heap sort | O(N log N) |
| 공간 | O(N) |

---

## 5. 언어별 구현 (11 언어)

### Python 3
```python
import heapq

h = []
heapq.heappush(h, 3)
heapq.heappush(h, 1)
heapq.heappush(h, 5)
print(heapq.heappop(h))   # 1

# Heapify in-place
nums = [3, 1, 4, 1, 5, 9, 2, 6]
heapq.heapify(nums)        # O(N)
print(nums[0])              # 1 (min)

# Max-heap → negate
heapq.heappush(h, -value)
print(-heapq.heappop(h))

# Top-K largest
top3 = heapq.nlargest(3, nums)  # [9, 6, 5]
top3 = heapq.nsmallest(3, nums) # [1, 1, 2]

# Priority Queue 패턴 — (priority, item)
heap = []
heapq.heappush(heap, (3, "task A"))
heapq.heappush(heap, (1, "task B"))
heapq.heappop(heap)   # (1, "task B")

# 직접 구현 (학습용)
class MinHeap:
    def __init__(self): self.data = []
    def push(self, x):
        self.data.append(x)
        self._sift_up(len(self.data) - 1)
    def pop(self):
        if not self.data: return None
        self._swap(0, len(self.data) - 1)
        result = self.data.pop()
        if self.data: self._sift_down(0)
        return result
    def _sift_up(self, i):
        while i > 0:
            parent = (i - 1) // 2
            if self.data[i] < self.data[parent]:
                self._swap(i, parent); i = parent
            else: break
    def _sift_down(self, i):
        n = len(self.data)
        while 2*i+1 < n:
            smallest = i
            if self.data[2*i+1] < self.data[smallest]: smallest = 2*i+1
            if 2*i+2 < n and self.data[2*i+2] < self.data[smallest]: smallest = 2*i+2
            if smallest == i: break
            self._swap(i, smallest); i = smallest
    def _swap(self, i, j): self.data[i], self.data[j] = self.data[j], self.data[i]
```

### Java
```java
import java.util.PriorityQueue;
import java.util.Collections;

PriorityQueue<Integer> minHeap = new PriorityQueue<>();
minHeap.offer(3); minHeap.offer(1);
minHeap.poll();   // 1

PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());

// 커스텀 비교
PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);
pq.offer(new int[]{3, 100});
pq.offer(new int[]{1, 200});
pq.poll();   // {1, 200}
```

### C++
```cpp
#include <queue>
#include <vector>

std::priority_queue<int> maxpq;   // max-heap (기본)
maxpq.push(3); maxpq.push(1);
maxpq.top();   // 3
maxpq.pop();

std::priority_queue<int, std::vector<int>, std::greater<int>> minpq;

// 커스텀 — pair (priority, item)
std::priority_queue<std::pair<int, int>, std::vector<std::pair<int,int>>,
                    std::greater<>> pq;

// make_heap / push_heap / pop_heap — 알고리즘 함수
std::vector<int> v = {3, 1, 4, 1, 5, 9};
std::make_heap(v.begin(), v.end());   // O(N)
```

### C — 직접 구현 (libc 에 없음)
```c
typedef struct {
    int data[10000];
    int size;
} MinHeap;

void swap(int *a, int *b) { int t = *a; *a = *b; *b = t; }

void sift_up(MinHeap *h, int i) {
    while (i > 0) {
        int p = (i - 1) / 2;
        if (h->data[i] < h->data[p]) { swap(&h->data[i], &h->data[p]); i = p; }
        else break;
    }
}

void push(MinHeap *h, int x) {
    h->data[h->size++] = x;
    sift_up(h, h->size - 1);
}

int pop(MinHeap *h) {
    int min = h->data[0];
    h->data[0] = h->data[--h->size];
    int i = 0;
    while (2*i+1 < h->size) {
        int s = i;
        if (h->data[2*i+1] < h->data[s]) s = 2*i+1;
        if (2*i+2 < h->size && h->data[2*i+2] < h->data[s]) s = 2*i+2;
        if (s == i) break;
        swap(&h->data[i], &h->data[s]); i = s;
    }
    return min;
}
```

### JavaScript
```javascript
// JS 표준에 없음 — 라이브러리: npm/heap-js, tinyqueue
// 직접 구현
class MinHeap {
    constructor() { this.data = []; }
    push(x) {
        this.data.push(x);
        this._siftUp(this.data.length - 1);
    }
    pop() {
        if (!this.data.length) return undefined;
        const top = this.data[0];
        const last = this.data.pop();
        if (this.data.length) { this.data[0] = last; this._siftDown(0); }
        return top;
    }
    _siftUp(i) {
        while (i > 0) {
            const p = (i - 1) >> 1;
            if (this.data[i] < this.data[p]) {
                [this.data[i], this.data[p]] = [this.data[p], this.data[i]];
                i = p;
            } else break;
        }
    }
    _siftDown(i) {
        const n = this.data.length;
        while (2*i+1 < n) {
            let s = i;
            if (this.data[2*i+1] < this.data[s]) s = 2*i+1;
            if (2*i+2 < n && this.data[2*i+2] < this.data[s]) s = 2*i+2;
            if (s === i) break;
            [this.data[i], this.data[s]] = [this.data[s], this.data[i]];
            i = s;
        }
    }
}
```

### TypeScript
```typescript
// 동일한 MinHeap 구조에 타입 추가
class MinHeap<T> {
    constructor(private cmp: (a: T, b: T) => number = (a, b) => (a as any) - (b as any)) {}
    private data: T[] = [];
    push(x: T): void {
        this.data.push(x); this._up(this.data.length - 1);
    }
    pop(): T | undefined {
        if (!this.data.length) return undefined;
        const top = this.data[0];
        const last = this.data.pop()!;
        if (this.data.length) { this.data[0] = last; this._down(0); }
        return top;
    }
    private _up(i: number): void { /* ... */ }
    private _down(i: number): void { /* ... */ }
}
```

### Kotlin
```kotlin
import java.util.PriorityQueue

val pq = PriorityQueue<Int>()
pq.offer(3); pq.offer(1)
pq.poll()    // 1

// 커스텀
val maxPq = PriorityQueue<Int>(compareByDescending { it })
val byScore = PriorityQueue<IntArray>(compareBy { it[0] })
```

### Swift
```swift
// Swift Algorithms 패키지에 Heap (Swift 5.7+)
// 외부 패키지 없으면 직접 구현
struct MinHeap<T: Comparable> {
    private var data: [T] = []
    var isEmpty: Bool { data.isEmpty }
    var top: T? { data.first }
    mutating func push(_ x: T) {
        data.append(x); siftUp(data.count - 1)
    }
    mutating func pop() -> T? {
        guard !data.isEmpty else { return nil }
        data.swapAt(0, data.count - 1)
        let top = data.removeLast()
        if !data.isEmpty { siftDown(0) }
        return top
    }
    // siftUp/siftDown 구현 생략
}
```

### Go
```go
package main

import "container/heap"

// container/heap 은 interface — 직접 구현 필요
type IntHeap []int
func (h IntHeap) Len() int { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int) { h[i], h[j] = h[j], h[i] }
func (h *IntHeap) Push(x any) { *h = append(*h, x.(int)) }
func (h *IntHeap) Pop() any { old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x }

// 사용
h := &IntHeap{2, 1, 5}
heap.Init(h)
heap.Push(h, 3)
heap.Pop(h)
```

### Rust
```rust
use std::collections::BinaryHeap;
use std::cmp::Reverse;

let mut max_heap = BinaryHeap::new();
max_heap.push(3); max_heap.push(1);
max_heap.pop();   // Some(3)

// Min-heap
let mut min_heap: BinaryHeap<Reverse<i32>> = BinaryHeap::new();
min_heap.push(Reverse(3));
min_heap.push(Reverse(1));
let Reverse(x) = min_heap.pop().unwrap();
```

### C#
```csharp
using System.Collections.Generic;

// .NET 6+ PriorityQueue<TElement, TPriority>
var pq = new PriorityQueue<string, int>();
pq.Enqueue("task1", 3);
pq.Enqueue("task2", 1);
pq.Dequeue();   // "task2"

// Min-priority 만 — 역순은 -priority 또는 Comparer
```

---

## 6. 변형 / 응용

### (a) Top-K — heap 크기 K 유지
```python
import heapq
def top_k(nums, k):
    heap = []
    for x in nums:
        heapq.heappush(heap, x)
        if len(heap) > k: heapq.heappop(heap)
    return sorted(heap, reverse=True)
```
공간 O(K), 시간 O(N log K).

### (b) Median Finder — 두 힙
left = max-heap (작은 절반), right = min-heap (큰 절반). 항상 균형.
```python
class MedianFinder:
    def __init__(self):
        self.lo = []   # max (negated)
        self.hi = []   # min
    def add(self, x):
        heapq.heappush(self.lo, -heapq.heappushpop(self.hi, x))
        if len(self.lo) > len(self.hi):
            heapq.heappush(self.hi, -heapq.heappop(self.lo))
    def median(self):
        if len(self.lo) < len(self.hi): return self.hi[0]
        return (self.hi[0] - self.lo[0]) / 2
```

### (c) Dijkstra — 최단 경로
[[../../algorithm/shortest-path/shortest-path]] 참조.

### (d) Huffman Coding
빈도 작은 두 노드 합치기 → 최적 인코딩.

### (e) Merge K sorted lists
각 리스트의 head 를 heap 에 → top 꺼내고 다음 push.
```python
def merge_k(lists):
    heap = [(l[0], i, 0) for i, l in enumerate(lists) if l]
    heapq.heapify(heap)
    result = []
    while heap:
        val, li, idx = heapq.heappop(heap)
        result.append(val)
        if idx + 1 < len(lists[li]):
            heapq.heappush(heap, (lists[li][idx+1], li, idx+1))
    return result
```

### (f) Event-driven simulation
시간 순 이벤트 큐.

### (g) Fibonacci Heap
- decrease-key O(1) amortized.
- Dijkstra/Prim 의 이론적 최적이지만 상수 큼 → 실무 X.

### (h) Pairing Heap
- 단순한 Fibonacci 대안. 실무에서 더 빠름.

---

## 7. 함정 / 안티패턴

### 함정 1 — Python heapq 는 min-heap 만
max-heap 필요 시 `-value` 또는 wrapper 클래스.

### 함정 2 — 임의 원소 삭제는 O(N)
정확한 위치 모르면 인덱스 hash 사용 또는 lazy deletion:
```python
class LazyHeap:
    def __init__(self): self.heap = []; self.removed = set()
    def push(self, x): heapq.heappush(self.heap, x)
    def pop(self):
        while self.heap and self.heap[0] in self.removed:
            self.removed.discard(heapq.heappop(self.heap))
        return heapq.heappop(self.heap)
    def remove(self, x): self.removed.add(x)
```

### 함정 3 — Heap 의 순회는 정렬 X
힙 내부 배열은 정렬 안 됨. 정렬된 출력 원하면 pop 반복.

### 함정 4 — JVM PriorityQueue 의 iterator 순서
순회 순서가 정렬 순서 아님. 정렬 원하면 pop 반복.

### 함정 5 — Build heap 을 N 번 push 로
O(N log N) — heapify 의 O(N) 보다 느림. 일괄 입력 시 `heapify(list)`.

### 함정 6 — 비교 함수 안정성
같은 우선순위 시 어떻게 처리? Python heapq 는 secondary key 로 tie-break.

### 함정 7 — Heap sort 가 quick sort 보다 느림
in-place 정렬이지만 캐시 비친화 — 실무 정렬은 Timsort / introsort.

### 함정 8 — Decimal / Float 비교
부동소수점 비교 시 epsilon. heap 의 비교 함수 일관성 깨지면 invariant 깨짐.

---

## 8. 실전 절차

1. **신호 인식** — "최솟값/최댓값 반복", "top-K", "스케줄링" → 힙.
2. **min vs max** — 언어별 wrapper 결정.
3. **size K 유지 패턴** — top-K 는 크기 K heap 유지로 O(N log K).
4. **lazy deletion** — 임의 원소 삭제 필요 시.

---

## 9. 대표 문제

### Q1. Kth Largest (LeetCode 215)
```python
import heapq
def find_kth_largest(nums, k):
    return heapq.nlargest(k, nums)[-1]
    # 또는 size K min-heap
```

### Q2. Top K Frequent (LeetCode 347)
```python
from collections import Counter
import heapq
def top_k(nums, k):
    return [x for x, _ in Counter(nums).most_common(k)]
```

### Q3. Merge K Sorted Lists (LeetCode 23)
위 (e) 참조.

### Q4. Find Median from Data Stream (LeetCode 295)
위 (b) 참조.

### Q5. Last Stone Weight (LeetCode 1046)
```python
import heapq
def last_stone(stones):
    h = [-x for x in stones]
    heapq.heapify(h)
    while len(h) > 1:
        a, b = -heapq.heappop(h), -heapq.heappop(h)
        if a != b: heapq.heappush(h, -(a - b))
    return -h[0] if h else 0
```

### Q6. Task Scheduler (LeetCode 621)
```python
from collections import Counter, deque
import heapq
def task_scheduler(tasks, n):
    counts = Counter(tasks)
    heap = [-c for c in counts.values()]
    heapq.heapify(heap)
    q = deque()   # (resume_time, count)
    time = 0
    while heap or q:
        time += 1
        if heap:
            cnt = heapq.heappop(heap) + 1
            if cnt: q.append((time + n, cnt))
        if q and q[0][0] == time:
            heapq.heappush(heap, q.popleft()[1])
    return time
```

---

## 10. 연습 문제

### 입문
- [LeetCode 215 Kth Largest](https://leetcode.com/problems/kth-largest-element-in-an-array/)
- [LeetCode 703 Kth Largest in Stream](https://leetcode.com/problems/kth-largest-element-in-a-stream/)
- [백준 11279 최대 힙](https://www.acmicpc.net/problem/11279)

### 중급
- [LeetCode 347 Top K Frequent](https://leetcode.com/problems/top-k-frequent-elements/)
- [LeetCode 23 Merge K Sorted Lists](https://leetcode.com/problems/merge-k-sorted-lists/)
- [LeetCode 1046 Last Stone Weight](https://leetcode.com/problems/last-stone-weight/)
- [백준 1655 가운데를 말해요](https://www.acmicpc.net/problem/1655)

### 고급
- [LeetCode 295 Find Median](https://leetcode.com/problems/find-median-from-data-stream/)
- [LeetCode 621 Task Scheduler](https://leetcode.com/problems/task-scheduler/)
- [LeetCode 502 IPO](https://leetcode.com/problems/ipo/)

---

## 11. 학습 자료

- **CLRS Ch.6** — Heaps and Heapsort
- **Sedgewick Algorithms Ch.2.4** — Priority Queues
- [VisuAlgo — Heap](https://visualgo.net/en/heap)
- [Build Heap is O(N) 증명](https://medium.com/@randerson112358/lets-build-a-min-heap-4d863cac6521)

---

## 12. 관련

- [[../trees/trees]] — 완전 이진 트리
- [[../arrays-and-strings/arrays-and-strings]] — 배열 표현
- [[../../algorithm/sorting/sorting]] — Heap Sort
- [[../../algorithm/shortest-path/shortest-path]] — Dijkstra 의 heap
- [[../data-structure|↑ data-structure]]
