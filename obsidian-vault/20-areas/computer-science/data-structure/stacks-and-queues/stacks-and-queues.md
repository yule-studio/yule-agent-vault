---
title: "스택 / 큐 (Stacks & Queues)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T06:30:00+09:00
tags:
  - data-structure
  - stack
  - queue
---

# 스택 / 큐 (Stacks & Queues)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 수준 + 11 언어 |

**[[../data-structure|↑ data-structure]]** · **[[../../computer-science|↑↑ computer-science]]**

---

## 1. 한 줄 정의

- **스택 (Stack)**: **LIFO** (Last In First Out). 가장 최근에 넣은 것이 가장 먼저 나옴.
- **큐 (Queue)**: **FIFO** (First In First Out). 가장 먼저 넣은 것이 가장 먼저 나옴.
- **덱 (Deque, Double-Ended Queue)**: 양 끝에서 삽입·삭제. 스택과 큐의 일반화.

모두 **접근 제한된 List ADT** — 특정 위치 (한 끝 / 양 끝) 에서만 연산 가능.

---

## 2. 개념의 깊이

### 2.1 왜 "제한된" 자료구조를 쓰는가

배열 / 리스트는 임의 위치 접근 가능 — 너무 자유로움.

**의도적 제한** 의 효과:
- 사용 패턴이 단순해 **버그 줄어듦**.
- 컴파일러 / 사람이 코드를 더 빨리 이해.
- 특정 알고리즘 (DFS, BFS, parsing) 의 자연스러운 모델.

### 2.2 스택의 본질 — 시간 역행

스택은 **시간을 거꾸로 되감는** 자료구조:
- 함수 호출 → 호출한 순서의 **역순**으로 반환 (call stack).
- Undo 명령 → 마지막 작업부터 되돌림.
- DFS → 깊이 우선 탐색 = 가장 최근 노드부터 계속 파고듦.

### 2.3 큐의 본질 — 도착 순서 보존

큐는 **공정성 (fairness)** 의 자료구조:
- 프린터 큐 — 먼저 인쇄 요청한 사람이 먼저.
- BFS — 가까운 노드를 먼저 처리 → 가중치 없는 그래프 최단 거리.
- OS 의 ready queue — 프로세스 스케줄링.
- 메시지 큐 (Kafka, RabbitMQ) — 분산 시스템의 핵심.

### 2.4 역사

- **1946 ENIAC** — 최초의 컴퓨터에 이미 함수 호출용 스택 개념.
- **1957 IBM 704** — 하드웨어 스택 레지스터.
- **1968 Dijkstra** — 큐가 BFS / scheduler 의 핵심.
- **1970s ALGOL/Pascal** — `procedure` 호출의 stack frame.

### 2.5 ADT 정의

**Stack**:
- `push(x)` — 맨 위에 추가
- `pop()` — 맨 위 제거 + 반환
- `top() / peek()` — 맨 위 조회 (제거 X)
- `is_empty()`, `size()`

**Queue**:
- `enqueue(x)` — 뒤에 추가
- `dequeue()` — 앞에서 제거 + 반환
- `front() / peek()` — 앞 조회
- `is_empty()`, `size()`

**Deque**:
- 위 모두 + `push_front`, `push_back`, `pop_front`, `pop_back`.

### 2.6 구현 옵션

| ADT | 배열 구현 | 연결 리스트 구현 |
| --- | --- | --- |
| Stack | ✅ 매우 자연스러움 — 끝에만 동작. amortized O(1). | ✅ 머리에 push/pop. O(1). |
| Queue | ⚠️ 단순 배열은 O(N) (한쪽이 shift). **Circular buffer** 로 O(1). | ✅ 머리와 꼬리 포인터. O(1). |
| Deque | **Block-based** (Python `deque`). | 이중 연결 리스트. |

---

## 3. 핵심 직관

### 스택 — 책 쌓기

```
 push(C)
 push(B)        ┌───┐
 push(A)        │ A │ ← top
                ├───┤
                │ B │
                ├───┤
                │ C │
                └───┘
 pop() → A (가장 최근 책)
```

### 큐 — 줄 서기

```
 enqueue: A → B → C  →  [A | B | C]
                          ↑       ↑
                        front   back
 dequeue() → A (가장 먼저 줄 선 사람)
```

---

## 4. Circular Buffer — 큐의 O(1) 구현

배열 기반 큐를 **고정 크기 + 환형** 으로:

```
크기 5:
  index:  0   1   2   3   4
  값:    [ ][a][b][c][ ]
         ↑          ↑
        head       tail

enqueue(d): tail += 1, mod 5
  값:    [ ][a][b][c][d]
                       ↑
                      tail
enqueue(e): tail = 0 (wraparound)
  값:    [e][a][b][c][d]
         ↑   ↑
        tail head
```

**조건**:
- 가득참: `(tail + 1) % size == head`
- 빔: `head == tail`

장점: O(1) 양 끝, 캐시 친화, 메모리 고정. 단점: 고정 크기.

---

## 5. 복잡도

| 연산 | Stack (배열) | Stack (LL) | Queue (배열) | Queue (Circular) | Queue (LL) | Deque (block) |
| --- | --- | --- | --- | --- | --- | --- |
| push / enqueue | O(1)* | O(1) | O(N) shift | **O(1)** | O(1) | **O(1)** |
| pop / dequeue | O(1) | O(1) | O(1) | **O(1)** | O(1) | **O(1)** |
| top / front | O(1) | O(1) | O(1) | O(1) | O(1) | O(1) |
| size | O(1) | O(1) | O(1) | O(1) | O(1) | O(1) |
| 공간 | O(N) | O(N) + 포인터 | O(N) | O(N) 고정 | O(N) + 포인터 | O(N) |

*amortized

---

## 6. 의사 코드

### 스택 (배열 기반)
```
push(x): a[size++] = x
pop():   return a[--size]
top():   return a[size - 1]
```

### 큐 (Circular Buffer)
```
enqueue(x):
    if full: resize or error
    a[tail] = x
    tail = (tail + 1) % capacity

dequeue():
    x = a[head]
    head = (head + 1) % capacity
    return x
```

---

## 7. 언어별 구현 (11 언어)

### Python 3
```python
# 스택 = list
stack = []
stack.append(1)            # push O(1) amortized
stack.append(2)
x = stack.pop()             # 2 — O(1)
top = stack[-1]             # 1

# 큐 = collections.deque (양 끝 O(1))
from collections import deque
queue = deque()
queue.append(1)             # enqueue (뒤)
queue.append(2)
x = queue.popleft()         # 1 — O(1) FIFO

# Deque (양 끝 사용)
dq = deque([1, 2, 3])
dq.appendleft(0)            # [0,1,2,3]
dq.pop()                    # [0,1,2]
dq.popleft()                # [1,2]

# queue.Queue — 스레드 안전 (느림)
import queue
q = queue.Queue()
q.put(1); q.get()

# heapq — 우선순위 큐
import heapq
pq = []
heapq.heappush(pq, 3)
heapq.heappush(pq, 1)
heapq.heappop(pq)           # 1
```

### Java
```java
import java.util.*;

// 스택 - Deque 권장 (Stack 클래스는 레거시)
Deque<Integer> stack = new ArrayDeque<>();
stack.push(1); stack.push(2);
int x = stack.pop();        // 2

// 큐 - ArrayDeque / LinkedList
Deque<Integer> queue = new ArrayDeque<>();
queue.offer(1); queue.offer(2);
int y = queue.poll();        // 1 FIFO

// PriorityQueue
PriorityQueue<Integer> pq = new PriorityQueue<>();
pq.offer(3); pq.offer(1);
pq.poll();                    // 1
```

### C++
```cpp
#include <stack>
#include <queue>
#include <deque>

std::stack<int> st;
st.push(1); st.push(2);
int x = st.top(); st.pop();

std::queue<int> q;
q.push(1); q.push(2);
int y = q.front(); q.pop();

std::deque<int> dq = {1, 2, 3};
dq.push_front(0); dq.pop_back();

// Priority queue (max-heap 기본)
std::priority_queue<int> pq;
pq.push(3); pq.push(1);
pq.top();   // 3

// Min-heap
std::priority_queue<int, std::vector<int>, std::greater<int>> minpq;
```

### C
```c
// 직접 구현 (배열 기반 스택)
typedef struct {
    int data[1000];
    int top;
} Stack;
void push(Stack *s, int x) { s->data[++s->top] = x; }
int pop(Stack *s) { return s->data[s->top--]; }
int empty(Stack *s) { return s->top == -1; }

// Circular Queue
typedef struct {
    int data[1000];
    int head, tail, size;
} Queue;
void enqueue(Queue *q, int x) {
    q->data[q->tail] = x;
    q->tail = (q->tail + 1) % 1000;
    q->size++;
}
int dequeue(Queue *q) {
    int x = q->data[q->head];
    q->head = (q->head + 1) % 1000;
    q->size--;
    return x;
}
```

### JavaScript
```javascript
// 스택 = 배열
const stack = [];
stack.push(1); stack.push(2);
stack.pop();        // 2

// 큐 — Array.shift() 는 O(N)
// 작은 입력 OK, 큰 입력은 직접 구현 (deque 클래스)
const queue = [];
queue.push(1); queue.push(2);
queue.shift();      // 1 — O(N) 주의

// 효율 큐 — head 인덱스 + 가끔 compact
class Queue {
    constructor() { this.items = []; this.head = 0; }
    enqueue(x) { this.items.push(x); }
    dequeue() {
        if (this.head >= this.items.length) return undefined;
        const x = this.items[this.head++];
        if (this.head > 1000) { this.items = this.items.slice(this.head); this.head = 0; }
        return x;
    }
}
```

### TypeScript
```typescript
class Stack<T> {
    private items: T[] = [];
    push(x: T): void { this.items.push(x); }
    pop(): T | undefined { return this.items.pop(); }
    top(): T | undefined { return this.items[this.items.length - 1]; }
}

class Queue<T> {
    private items: T[] = [];
    private head = 0;
    enqueue(x: T): void { this.items.push(x); }
    dequeue(): T | undefined {
        return this.head < this.items.length ? this.items[this.head++] : undefined;
    }
}
```

### Kotlin
```kotlin
import java.util.ArrayDeque
import java.util.PriorityQueue

val stack = ArrayDeque<Int>()
stack.push(1); stack.push(2)
stack.pop()                     // 2

val queue = ArrayDeque<Int>()
queue.offer(1); queue.offer(2)
queue.poll()                    // 1

val pq = PriorityQueue<Int>()
pq.offer(3); pq.offer(1)
pq.poll()                       // 1
```

### Swift
```swift
// Swift 표준에 Stack/Queue 없음 — Array 또는 직접
var stack = [Int]()
stack.append(1)
let x = stack.popLast()         // O(1)

// 큐는 효율 위해 두 스택 사용 또는 직접 구현
struct Queue<T> {
    private var inbox: [T] = []
    private var outbox: [T] = []
    mutating func enqueue(_ x: T) { inbox.append(x) }
    mutating func dequeue() -> T? {
        if outbox.isEmpty { outbox = inbox.reversed(); inbox = [] }
        return outbox.popLast()
    }
}
```

### Go
```go
package main

import "container/heap"

// 스택 = slice
stack := []int{}
stack = append(stack, 1)
n := len(stack)
x := stack[n-1]
stack = stack[:n-1]

// 큐 = container/list 또는 slice (작은 입력)
import "container/list"
q := list.New()
q.PushBack(1); q.PushBack(2)
front := q.Front()
q.Remove(front)

// 최소 힙 — container/heap interface 구현 필요
type IntHeap []int
func (h IntHeap) Len() int { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int) { h[i], h[j] = h[j], h[i] }
func (h *IntHeap) Push(x any) { *h = append(*h, x.(int)) }
func (h *IntHeap) Pop() any { old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x }
```

### Rust
```rust
// 스택 = Vec
let mut stack: Vec<i32> = Vec::new();
stack.push(1); stack.push(2);
let x = stack.pop();              // Some(2)

// 큐 = VecDeque
use std::collections::VecDeque;
let mut q: VecDeque<i32> = VecDeque::new();
q.push_back(1); q.push_back(2);
let y = q.pop_front();            // Some(1)

// 최대 힙 = BinaryHeap
use std::collections::BinaryHeap;
let mut pq = BinaryHeap::new();
pq.push(3); pq.push(1);
pq.pop();                          // Some(3)
// Min-heap: BinaryHeap<Reverse<i32>>
```

### C#
```csharp
using System.Collections.Generic;

var stack = new Stack<int>();
stack.Push(1); stack.Push(2);
int x = stack.Pop();

var queue = new Queue<int>();
queue.Enqueue(1); queue.Enqueue(2);
int y = queue.Dequeue();

// PriorityQueue (.NET 6+)
var pq = new PriorityQueue<string, int>();
pq.Enqueue("task1", 3);
pq.Enqueue("task2", 1);
var top = pq.Dequeue();           // "task2"
```

---

## 8. 변형 / 응용

### (a) Monotonic Stack — 다음 큰 원소
"오른쪽에서 자기보다 큰 첫 원소" 찾기 O(N).

```python
def next_greater(arr):
    result = [-1] * len(arr)
    stack = []
    for i, x in enumerate(arr):
        while stack and arr[stack[-1]] < x:
            result[stack.pop()] = x
        stack.append(i)
    return result
```

### (b) Min Stack — 최솟값을 O(1) 에 조회
보조 스택에 누적 최솟값 저장.

### (c) Queue using 2 Stacks
inbox + outbox. dequeue 시 outbox 비면 inbox 의 모든 걸 reverse 해서 옮김. amortized O(1).

### (d) Stack using 2 Queues
queue1 에서 enqueue, dequeue 시 queue1 의 마지막 원소 빼고 queue2 로 옮김.

### (e) Sliding Window Maximum — Deque
```python
from collections import deque
def max_sliding(nums, k):
    dq = deque()
    out = []
    for i, x in enumerate(nums):
        while dq and nums[dq[-1]] < x: dq.pop()
        dq.append(i)
        if dq[0] == i - k: dq.popleft()
        if i >= k - 1: out.append(nums[dq[0]])
    return out
```

### (f) Priority Queue — 우선순위 큐
힙으로 구현. Dijkstra / 작업 스케줄링 / top-K.

### (g) Circular Queue / Ring Buffer
임베디드 / OS 의 IO buffer.

### (h) BFS — 큐의 대표 응용
가장 큰 시그널: **큐 보이면 BFS, 스택 보이면 DFS**.

---

## 9. 함정 / 안티패턴

### 함정 1 — `list.pop(0)` is O(N)
```python
# 잘못
queue = []
queue.append(1)
queue.pop(0)        # O(N) — 큰 입력에서 TLE

# 옳음
from collections import deque
queue = deque()
queue.append(1)
queue.popleft()     # O(1)
```

### 함정 2 — JavaScript `Array.shift()` 도 O(N)
큰 큐는 head 인덱스 패턴 또는 라이브러리 사용.

### 함정 3 — Java `Stack` 레거시
`Stack` 클래스 (Vector 상속) 은 동기화 + 느림. `Deque<>` 사용.

### 함정 4 — Circular Queue 의 full / empty 구분
같은 조건 (head == tail) 이 둘 다 가능. **size 카운터** 또는 **한 슬롯 비워두기**.

### 함정 5 — Stack overflow 재귀
DFS 깊이 큰 그래프에서 재귀 → 스택 오버플로. 반복문 + 명시적 스택으로.

### 함정 6 — Python `queue.Queue` 의 성능
스레드 안전 (lock 사용) → 단일 스레드 코테에서는 느림. `deque` 사용.

### 함정 7 — 우선순위 큐의 정렬 기준
파이썬 `heapq` 는 min-heap. max-heap 원하면 `-value` 또는 wrapper.

### 함정 8 — Stack 의 top 가 변경되지 않는다고 가정
`stack[-1]` 만 보고 `pop` 안 하면 같은 값 반복 처리 가능.

---

## 10. 실전 절차

1. **신호 인식** — "괄호 짝", "이전 큰 값", "BFS", "DFS", "취소" → 스택/큐 후보.
2. **자료구조 선택** — 단일 끝 → 스택, 양 끝 → deque, 우선순위 → 힙.
3. **언어 내장 사용** — 거의 항상 내장 (Python `deque`, Java `ArrayDeque`, C++ `stack/queue`).
4. **재귀 vs 명시 스택** — 깊이 10^5 이상이면 명시.

---

## 11. 대표 문제

### Q1. 유효한 괄호 (LeetCode 20)
```python
def is_valid(s):
    stack = []
    pairs = {')': '(', '}': '{', ']': '['}
    for c in s:
        if c in '({[': stack.append(c)
        else:
            if not stack or stack.pop() != pairs[c]: return False
    return not stack
```

### Q2. 매일 더 따뜻한 날 (LeetCode 739) — Monotonic Stack
```python
def daily_temps(temps):
    result = [0] * len(temps)
    stack = []
    for i, t in enumerate(temps):
        while stack and temps[stack[-1]] < t:
            j = stack.pop()
            result[j] = i - j
        stack.append(i)
    return result
```

### Q3. Min Stack (LeetCode 155)
```python
class MinStack:
    def __init__(self):
        self.stack = []
        self.min_stack = []
    def push(self, x):
        self.stack.append(x)
        if not self.min_stack or x <= self.min_stack[-1]:
            self.min_stack.append(x)
    def pop(self):
        if self.stack.pop() == self.min_stack[-1]:
            self.min_stack.pop()
    def getMin(self): return self.min_stack[-1]
```

### Q4. 두 스택으로 큐 구현 (LeetCode 232)
```python
class MyQueue:
    def __init__(self):
        self.inbox = []
        self.outbox = []
    def push(self, x): self.inbox.append(x)
    def pop(self):
        self.peek()
        return self.outbox.pop()
    def peek(self):
        if not self.outbox:
            while self.inbox: self.outbox.append(self.inbox.pop())
        return self.outbox[-1]
    def empty(self): return not self.inbox and not self.outbox
```

### Q5. Sliding Window Maximum (LeetCode 239) — 위 코드

### Q6. 후위 표기식 계산 (Postfix)
```python
def eval_postfix(tokens):
    stack = []
    for t in tokens:
        if t in '+-*/':
            b, a = stack.pop(), stack.pop()
            stack.append(int(eval(f'{a}{t}{b}')))
        else: stack.append(int(t))
    return stack[0]
```

---

## 12. 연습 문제

### 입문
- [백준 10828 스택](https://www.acmicpc.net/problem/10828)
- [백준 18258 큐 2](https://www.acmicpc.net/problem/18258)
- [LeetCode 20 Valid Parentheses](https://leetcode.com/problems/valid-parentheses/)

### 중급
- [LeetCode 232 Queue using Stacks](https://leetcode.com/problems/implement-queue-using-stacks/)
- [LeetCode 155 Min Stack](https://leetcode.com/problems/min-stack/)
- [LeetCode 739 Daily Temperatures](https://leetcode.com/problems/daily-temperatures/)
- [백준 17298 오큰수](https://www.acmicpc.net/problem/17298) — Monotonic Stack

### 고급
- [LeetCode 84 Largest Rectangle in Histogram](https://leetcode.com/problems/largest-rectangle-in-histogram/) — Monotonic
- [LeetCode 239 Sliding Window Maximum](https://leetcode.com/problems/sliding-window-maximum/)
- [LeetCode 224 Basic Calculator](https://leetcode.com/problems/basic-calculator/) — 스택 + 파싱

---

## 13. 학습 자료

- CLRS Ch.10
- [VisuAlgo — Stack/Queue/Deque](https://visualgo.net/en/list)
- [동빈나 — 스택/큐](https://www.youtube.com/@dongbinna)
- "윤성우의 열혈 자료구조" (한국어 직접 구현)
- Python `collections.deque` 내부 — block-based linked list
- Java `ArrayDeque` 소스

---

## 14. 관련

- [[../arrays-and-strings/arrays-and-strings]] — 스택의 배열 구현
- [[../linked-lists/linked-lists]] — 큐의 연결 리스트 구현
- [[../heaps/heaps]] — 우선순위 큐
- [[../../algorithm/dfs-bfs/dfs-bfs]] — 스택 → DFS, 큐 → BFS
- [[../data-structure|↑ data-structure]]
