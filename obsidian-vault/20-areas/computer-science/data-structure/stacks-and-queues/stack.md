---
title: "Stack — LIFO"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T13:00:00+09:00
tags:
  - data-structure
  - stack
---

# Stack — LIFO

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | LIFO / 구현 / 응용 |

**[[stacks-and-queues|↑ Stacks & Queues]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

**Last-In-First-Out** — 마지막 push 가 첫 pop. 함수 호출 / undo / parsing / DFS.

---

## 2. 연산

| | 설명 | 시간 |
| --- | --- | --- |
| `push(x)` | 위에 추가 | O(1) |
| `pop()` | 위 제거 + 반환 | O(1) |
| `peek/top()` | 위 보기 | O(1) |
| `empty()` | 비어있나 | O(1) |
| `size()` | 크기 | O(1) |

---

## 3. 구현

### Array 기반
```python
class Stack:
    def __init__(self):
        self.data = []
    
    def push(self, x):
        self.data.append(x)
    
    def pop(self):
        return self.data.pop() if self.data else None
    
    def peek(self):
        return self.data[-1] if self.data else None
```

### Linked list 기반
```python
class Stack:
    def __init__(self):
        self.head = None
    
    def push(self, x):
        self.head = Node(x, self.head)
    
    def pop(self):
        if not self.head: return None
        val = self.head.val
        self.head = self.head.next
        return val
```

### 비교
- Array — cache locality 좋음, resize 시 O(N)
- Linked list — O(1) 보장, but 메모리 ↑

---

## 4. 언어별

### Python
```python
stack = []
stack.append(1)   # push
stack.append(2)
stack.pop()       # 2 (LIFO)
stack[-1]         # peek
```

### Java
```java
Deque<Integer> stack = new ArrayDeque<>();    // 권장
stack.push(1);
stack.pop();
stack.peek();

// Stack class — 옛, Vector 기반 (synchronized 부담)
// ArrayDeque 권장
```

### C++
```cpp
std::stack<int> s;
s.push(1);
s.pop();          // remove (no return)
int top = s.top();    // separate
```

### Go
```go
stack := []int{}
stack = append(stack, 1)         // push
v := stack[len(stack)-1]          // peek
stack = stack[:len(stack)-1]     // pop
```

### Rust
```rust
let mut stack: Vec<i32> = Vec::new();
stack.push(1);
stack.pop();        // Option<i32>
stack.last();       // Option<&i32>
```

---

## 5. 응용 — 핵심

### 5.1 함수 호출 stack
- Call stack — caller / callee 의 frame
- Return address / 로컬 변수
- Recursion depth limit

### 5.2 Undo / Redo
- Undo — pop action stack
- Redo — pop redo stack
- 새 action — redo stack clear

### 5.3 Balanced parentheses
```python
def is_valid(s):
    stack = []
    pairs = {')': '(', ']': '[', '}': '{'}
    for c in s:
        if c in '([{':
            stack.append(c)
        elif c in ')]}':
            if not stack or stack[-1] != pairs[c]:
                return False
            stack.pop()
    return not stack
```

### 5.4 DFS — Iterative
```python
def dfs(graph, start):
    visited = set()
    stack = [start]
    while stack:
        node = stack.pop()
        if node in visited: continue
        visited.add(node)
        for nei in graph[node]:
            stack.append(nei)
```

### 5.5 Expression evaluation (RPN — postfix)
```python
def eval_rpn(tokens):
    stack = []
    for t in tokens:
        if t in '+-*/':
            b = stack.pop()
            a = stack.pop()
            if t == '+': stack.append(a + b)
            elif t == '-': stack.append(a - b)
            elif t == '*': stack.append(a * b)
            else: stack.append(int(a / b))
        else:
            stack.append(int(t))
    return stack[0]
```

### 5.6 Infix → Postfix (Shunting Yard)
- Operator precedence
- Dijkstra 알고리즘

### 5.7 Backtracking
- 상태 push → 실패 → pop → 다른 시도

---

## 6. Monotonic Stack — 패턴

### 정의
- Stack 안 — 단조 (증가 / 감소)
- 새 원소가 단조 깨면 — 깨는 동안 pop

### Daily Temperatures (다음 더 큰 원소)
```python
def daily_temperatures(T):
    result = [0] * len(T)
    stack = []      # indices, decreasing temps
    for i, t in enumerate(T):
        while stack and T[stack[-1]] < t:
            j = stack.pop()
            result[j] = i - j
        stack.append(i)
    return result
```

### Largest Rectangle in Histogram
```python
def largest_rectangle(heights):
    stack = []
    max_area = 0
    heights.append(0)    # sentinel
    for i, h in enumerate(heights):
        while stack and heights[stack[-1]] > h:
            top = stack.pop()
            width = i if not stack else i - stack[-1] - 1
            max_area = max(max_area, heights[top] * width)
        stack.append(i)
    return max_area
```

자세히 → [[monotonic-stack-deque]]

---

## 7. Min Stack — O(1) min

### 문제
- Stack 의 모든 시점의 min 을 O(1)

### 해결 — 2 stack
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
        x = self.stack.pop()
        if x == self.min_stack[-1]:
            self.min_stack.pop()
    
    def get_min(self):
        return self.min_stack[-1]
```

---

## 8. 두 stack 으로 queue

### MyQueue
```python
class MyQueue:
    def __init__(self):
        self.in_stack = []
        self.out_stack = []
    
    def push(self, x):
        self.in_stack.append(x)
    
    def pop(self):
        self._move()
        return self.out_stack.pop()
    
    def peek(self):
        self._move()
        return self.out_stack[-1]
    
    def _move(self):
        if not self.out_stack:
            while self.in_stack:
                self.out_stack.append(self.in_stack.pop())
```

→ amortized O(1).

---

## 9. 함정

### 함정 1 — 빈 stack pop
None / exception. 체크.

### 함정 2 — Recursion 의 stack overflow
큰 입력 — iterative + explicit stack.

### 함정 3 — Java `Stack` 의 synchronized
옛 클래스 — 모든 method synchronized. ArrayDeque 권장.

### 함정 4 — Linked list 의 cache miss
큰 throughput — array 기반.

### 함정 5 — Monotonic 의 등호
`<` vs `<=` — 문제 따라 결정.

### 함정 6 — Iterator 사용 X
Stack 은 일반적으로 iteration 안 함 — 순서 의미가 LIFO 이라서.

---

## 10. 학습 자료

- CLRS Ch 10
- LeetCode Stack tag
- "Dijkstra's Shunting Yard Algorithm"

---

## 11. 관련

- [[stacks-and-queues]] — Hub
- [[queue]] — 비교 (FIFO)
- [[monotonic-stack-deque]] — 패턴
- [[../../algorithm/algorithm]] — DFS / 표현식
