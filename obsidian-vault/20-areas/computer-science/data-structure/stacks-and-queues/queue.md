---
title: "Queue — FIFO"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T13:05:00+09:00
tags:
  - data-structure
  - queue
---

# Queue — FIFO

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | FIFO / circular / BFS |

**[[stacks-and-queues|↑ Stacks & Queues]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

**First-In-First-Out** — 먼저 들어온 것 먼저 나옴. BFS / scheduling / printer / 메시지 큐.

---

## 2. 연산

| | 설명 | 시간 |
| --- | --- | --- |
| `enqueue(x)` | 뒤에 추가 | O(1) |
| `dequeue()` | 앞에서 제거 | O(1) |
| `front()` | 앞 보기 | O(1) |
| `empty()` | 비어있나 | O(1) |
| `size()` | 크기 | O(1) |

---

## 3. 구현

### Circular Buffer (Array 기반)
```python
class CircularQueue:
    def __init__(self, k):
        self.data = [0] * k
        self.head = 0
        self.tail = 0
        self.size = 0
        self.cap = k
    
    def enqueue(self, x):
        if self.size == self.cap: return False
        self.data[self.tail] = x
        self.tail = (self.tail + 1) % self.cap
        self.size += 1
        return True
    
    def dequeue(self):
        if self.size == 0: return None
        x = self.data[self.head]
        self.head = (self.head + 1) % self.cap
        self.size -= 1
        return x
```

#### 장점
- O(1) 보장
- Cache locality

#### 단점
- 고정 크기

### Linked List 기반
```python
class Queue:
    def __init__(self):
        self.head = None
        self.tail = None
    
    def enqueue(self, x):
        node = Node(x)
        if not self.head:
            self.head = self.tail = node
        else:
            self.tail.next = node
            self.tail = node
    
    def dequeue(self):
        if not self.head: return None
        x = self.head.val
        self.head = self.head.next
        if not self.head: self.tail = None
        return x
```

### Naive — array (X)
```python
q = []
q.append(1)
q.pop(0)        # O(N)!!  — 모든 원소 shift
```

→ 절대 X. `deque` 권장.

---

## 4. 언어별

### Python `collections.deque`
```python
from collections import deque

q = deque()
q.append(1)         # enqueue
q.popleft()         # dequeue O(1)
q[0]                # front

# 양 끝 — O(1)
# Middle access — O(N)
```

### Java `Queue` interface
```java
Queue<Integer> q = new LinkedList<>();
// 또는
Queue<Integer> q = new ArrayDeque<>();

q.offer(1);    // enqueue (true on success)
q.poll();      // dequeue (null on empty)
q.peek();      // front
```

### C++
```cpp
std::queue<int> q;
q.push(1);
q.pop();        // void
int x = q.front();
```

### Go
```go
// container/list (linked) 또는 slice
q := []int{}
q = append(q, 1)         // enqueue
x := q[0]                 // front
q = q[1:]                 // dequeue (slice — 메모리 leak 가능)
```

### Rust
```rust
use std::collections::VecDeque;

let mut q: VecDeque<i32> = VecDeque::new();
q.push_back(1);
q.pop_front();
```

---

## 5. 응용

### 5.1 BFS — Breadth First Search
```python
from collections import deque

def bfs(graph, start):
    visited = {start}
    q = deque([start])
    while q:
        node = q.popleft()
        for nei in graph[node]:
            if nei not in visited:
                visited.add(nei)
                q.append(nei)
```

자세히 → algorithm/graph/bfs

### 5.2 Level-order traversal (트리)
```python
def level_order(root):
    if not root: return []
    result = []
    q = deque([root])
    while q:
        level = []
        for _ in range(len(q)):
            node = q.popleft()
            level.append(node.val)
            if node.left: q.append(node.left)
            if node.right: q.append(node.right)
        result.append(level)
    return result
```

### 5.3 OS — Scheduling
- Process queue (round-robin)
- Printer queue
- Network packet queue

### 5.4 Message queue
- Producer-Consumer
- Async task queue
- RabbitMQ / Kafka

자세히 → [[../../network/rpc-messaging/mqtt-amqp-kafka]]

### 5.5 Cache (FIFO eviction)
- 먼저 들어온 것 evict
- LRU 가 보통 더 좋지만 단순

### 5.6 Sliding window
- Window 의 원소 — queue
- 새 원소 enqueue, 나가는 원소 dequeue

---

## 6. Priority Queue

### 정의
- 우선순위 기반 dequeue (FIFO 아님)
- 보통 heap 구현

자세히 → [[../heaps/heaps]]

---

## 7. Blocking Queue (멀티 스레드)

### 정의
- Producer 가 가득이면 — block
- Consumer 가 비어있으면 — block

### Java
```java
BlockingQueue<Integer> q = new LinkedBlockingQueue<>(100);
q.put(1);        // blocking enqueue
int x = q.take();  // blocking dequeue
```

### Python
```python
from queue import Queue

q = Queue(maxsize=100)
q.put(1)         # blocking
x = q.get()      # blocking
```

---

## 8. Concurrent Queue

### Lock-free queue (Michael-Scott)
- CAS 기반
- 매우 빠름 / 복잡

### Disruptor (LMAX)
- Ring buffer + sequence
- 매우 빠름 (수천만 ops/sec)
- 금융 거래

### Go channel
- Built-in concurrent queue
- Buffered / unbuffered

---

## 9. Deque (Double-ended)

### 정의
- 양 끝 push / pop 둘 다

자세히 → [[deque]]

---

## 10. 함정

### 함정 1 — `list.pop(0)` 의 O(N)
Python list — 앞 pop 은 모든 원소 shift. `deque` 권장.

### 함정 2 — Go slice 의 dequeue
`q = q[1:]` — 메모리 leak (backing array 의 head 부분).

### 함정 3 — Circular 의 full / empty 구분
size 별도 또는 한 칸 비움. Tail==head — empty vs full?

### 함정 4 — Java `Queue` 의 `add` vs `offer`
add — 실패 시 exception. offer — false.

### 함정 5 — `peek` vs `front`
언어 별 — 같은 의미.

### 함정 6 — Thread safety
멀티 thread — concurrent queue / lock.

---

## 11. 학습 자료

- CLRS Ch 10
- LeetCode BFS / Queue tag
- "Java Concurrency in Practice" — BlockingQueue
- LMAX Disruptor docs

---

## 12. 관련

- [[stacks-and-queues]] — Hub
- [[stack]] — 비교 (LIFO)
- [[deque]] — 양 끝
- [[../heaps/heaps]] — Priority Queue
- [[../../algorithm/algorithm]] — BFS
