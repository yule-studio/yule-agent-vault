---
title: "Deque — Double-Ended Queue"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T13:10:00+09:00
tags:
  - data-structure
  - deque
---

# Deque — Double-Ended Queue

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 양쪽 push/pop O(1) |

**[[stacks-and-queues|↑ Stacks & Queues]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

**양 끝에서 push / pop** 모두 O(1). Stack + Queue 의 일반화. Sliding window / Monotonic deque 의 기반.

---

## 2. 연산

| | 시간 |
| --- | --- |
| `push_front(x)` | O(1) |
| `push_back(x)` | O(1) |
| `pop_front()` | O(1) |
| `pop_back()` | O(1) |
| `front() / back()` | O(1) |
| `size()` | O(1) |

---

## 3. 구현

### Python `collections.deque`
```python
from collections import deque

dq = deque()
dq.append(1)         # back
dq.appendleft(0)     # front
dq.pop()             # back
dq.popleft()         # front
dq[0]                # front
dq[-1]               # back
```

### 내부 구현 — block doubly linked
- 각 block — 약 64 원소
- Block 들의 doubly linked list
- O(1) 양 끝 + 적당한 cache (full array 보다는 못함)

### Java `ArrayDeque`
```java
Deque<Integer> dq = new ArrayDeque<>();
dq.offerFirst(1);
dq.offerLast(2);
dq.pollFirst();
dq.pollLast();
dq.peekFirst();
dq.peekLast();
```

### 내부 — Circular buffer (array 기반)
- Resize 시 2배
- 매우 빠름 (LinkedList 보다)

### C++ `std::deque`
```cpp
std::deque<int> dq;
dq.push_front(1);
dq.push_back(2);
dq.pop_front();
dq.pop_back();
dq.front();
dq.back();
```

### 내부 — 일반적으로 multiple fixed-size blocks
- 직접 indexing O(1)
- 약간의 indirection

### Rust `VecDeque`
- Ring buffer (single contiguous array)
- 매우 빠름
- 양 끝 모두 amortized O(1)

### Go (직접)
- 표준 deque 없음
- container/list (linked) 또는 직접 ring buffer

---

## 4. Deque vs Linked List

| | Deque (array-based) | Linked List |
| --- | --- | --- |
| **공간** | 작음 (포인터 없음) | 큼 (포인터) |
| **Cache** | 좋음 | 나쁨 |
| **Resize** | 가끔 (amortized) | 없음 |
| **Random access** | O(1) | O(N) |
| **사용** | 대부분 권장 | drop-in for list |

---

## 5. 응용 — 핵심

### 5.1 Sliding Window Max
```python
from collections import deque

def max_window(arr, k):
    dq = deque()       # indices, decreasing values
    result = []
    for i, v in enumerate(arr):
        while dq and dq[0] <= i - k:
            dq.popleft()        # 윈도우 벗어남
        while dq and arr[dq[-1]] < v:
            dq.pop()             # 더 작은 — 의미 없음
        dq.append(i)
        if i >= k - 1:
            result.append(arr[dq[0]])
    return result
```

자세히 → [[monotonic-stack-deque]]

### 5.2 0-1 BFS
- BFS 에서 — 가중치 0 / 1 edge 만
- Deque — 0 edge 면 push_front, 1 edge 면 push_back
- 결과 — Dijkstra 와 같은 결과 O(V+E)

### 5.3 Browser back/forward
- Back stack + forward stack
- 또는 — deque 의 중앙 pointer (drop after)

### 5.4 Round-robin scheduler
- Deque + rotate
- Take front, process, push_back

### 5.5 Steal queue (work stealing)
- 자기 thread — 한쪽
- 다른 thread 가 steal — 반대쪽
- Lock contention ↓

---

## 6. Monotonic Deque

### 정의
- Deque 안 — 단조 (sliding 의 boundary 별)
- 새 원소 추가 시 — 단조 깨는 것 pop_back

### 사용
- Sliding window min/max
- 0-1 BFS

자세히 → [[monotonic-stack-deque]]

---

## 7. Python `collections.deque` 의 옵션

### `maxlen`
```python
dq = deque(maxlen=3)
dq.append(1)
dq.append(2)
dq.append(3)
dq.append(4)
# dq == deque([2, 3, 4], maxlen=3)
```

→ Ring buffer 효과 (오래된 자동 제거).

### `rotate`
```python
dq = deque([1, 2, 3, 4, 5])
dq.rotate(2)    # [4, 5, 1, 2, 3]
dq.rotate(-1)   # [5, 1, 2, 3, 4]
```

---

## 8. 함정

### 함정 1 — Random access 비용
Python deque — `dq[5]` O(N). 양 끝만 O(1).

### 함정 2 — Iteration 의 효율
Linked-based — cache miss. Array-based — 좋음.

### 함정 3 — Java `LinkedList`
deque 인터페이스 구현 — but 진짜 linked. ArrayDeque 권장.

### 함정 4 — Concurrent X
대부분 thread-unsafe. ConcurrentLinkedDeque (Java) 등.

### 함정 5 — Memory 의 implicit growth
Python deque — block 마다 64 원소 — 작은 N 도 큰 메모리.

### 함정 6 — Rotate 의 O(N)
큰 rotation — 모든 원소 이동.

---

## 9. 학습 자료

- Python `collections` docs
- Java `ArrayDeque` docs
- "Java Concurrency in Practice" — work stealing

---

## 10. 관련

- [[stacks-and-queues]] — Hub
- [[stack]] / [[queue]] — 특수
- [[monotonic-stack-deque]] — 패턴
- [[../linked-lists/doubly-linked-list]] — 대안 구현
