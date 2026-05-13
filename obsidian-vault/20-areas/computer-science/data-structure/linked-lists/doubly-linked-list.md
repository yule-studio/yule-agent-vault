---
title: "Doubly Linked List — 이중 연결 리스트"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T12:05:00+09:00
tags:
  - data-structure
  - linked-list
---

# Doubly Linked List

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | next + prev / 응용 |

**[[linked-lists|↑ Linked Lists]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

각 노드가 **next + prev** 둘 다 가진 linked list. 양방향 순회 / O(1) 노드 삭제 가능.

---

## 2. 구조

```
NULL ← [1] ⇄ [2] ⇄ [3] ⇄ [4] → NULL
       head                tail
```

### 노드
```python
class Node:
    def __init__(self, val):
        self.val = val
        self.prev = None
        self.next = None
```

---

## 3. 장점 vs 단점

### Singly 대비 장점
- 역방향 순회
- 노드 자체 가지면 — O(1) 삭제 (prev 도 알아서)
- Deque 자연스러움

### 단점
- 메모리 ↑ (pointer 2 개)
- 코드 복잡 (prev/next 둘 다 update)

---

## 4. 연산

### Insert at head
```python
def insert_head(head, val):
    node = Node(val)
    if head:
        node.next = head
        head.prev = node
    return node
```

### Insert at tail
```python
def insert_tail(tail, val):
    node = Node(val)
    if tail:
        tail.next = node
        node.prev = tail
    return node
```

### Insert after given node
```python
def insert_after(node, val):
    new_node = Node(val)
    new_node.prev = node
    new_node.next = node.next
    if node.next:
        node.next.prev = new_node
    node.next = new_node
```

### Delete given node — O(1)
```python
def delete(node):
    if node.prev:
        node.prev.next = node.next
    if node.next:
        node.next.prev = node.prev
```

→ Singly 와 차이 — head 부터 탐색 X.

---

## 5. Sentinel 활용

```
[HEAD] ⇄ [a] ⇄ [b] ⇄ [c] ⇄ [TAIL]
```

### 효과
- 빈 list 도 head/tail node 존재
- Boundary 처리 단순
- Insert/delete 가 항상 같은 코드

```python
class DLL:
    def __init__(self):
        self.head = Node(0)
        self.tail = Node(0)
        self.head.next = self.tail
        self.tail.prev = self.head
    
    def add_first(self, val):
        return self.insert_after(self.head, val)
    
    def remove_last(self):
        if self.tail.prev != self.head:
            self.delete(self.tail.prev)
```

---

## 6. 응용

### 6.1 LRU Cache
자세히 → [[lru-cache]]

- Hash map + Doubly Linked List
- Access 시 — node 를 head 로 이동 (O(1))
- 가득 시 — tail 삭제 (O(1))

### 6.2 Deque (Java LinkedList)
- 양 끝 push / pop — O(1)
- Java `LinkedList` / Go `container/list`

### 6.3 Undo / Redo
- 양방향 — 이전 / 다음 state

### 6.4 Browser history
- 뒤로 / 앞으로

### 6.5 OS — Process list
- Linux task_struct

---

## 7. Circular Doubly Linked List

```
   ┌────────────────────────┐
   ↓                        │
[a] ⇄ [b] ⇄ [c] ⇄ [d] ─────┘
   ↑                        │
   └────────────────────────┘
```

### 효과
- Head/tail 같은 곳 (없음)
- O(1) head/tail access (any node 부터)

### 사용
- Round-robin scheduler
- Buffer / queue

---

## 8. 구현 — Java `LinkedList`

```java
LinkedList<Integer> list = new LinkedList<>();

// Deque 인터페이스
list.addFirst(1);
list.addLast(2);
list.removeFirst();
list.removeLast();

// Iterator
ListIterator<Integer> it = list.listIterator();
while (it.hasNext()) {
    int x = it.next();
    if (x == 3) it.remove();    // O(1) 삭제
}
```

### 한계
- 임의 index access — O(N)
- 일반 사용 — ArrayList 권장

---

## 9. Python `collections.deque`

```python
from collections import deque

dq = deque()
dq.append(1)        # O(1) right
dq.appendleft(2)    # O(1) left
dq.pop()            # O(1)
dq.popleft()        # O(1)

dq[0]               # O(1)
dq[5]               # O(N) (실제 block 기반)
```

### 내부
- doubly linked list 아니라 — block (array chunks) 의 doubly linked
- O(1) 양 끝 + 적당한 cache locality

---

## 10. 함정

### 함정 1 — prev/next 의 동시 update
한쪽만 update — 망가짐. 두 방향 모두 갱신.

### 함정 2 — Sentinel 없이 boundary
빈 list / 첫 / 마지막 — 특수 처리. Sentinel 권장.

### 함정 3 — 중간 노드 삭제 시 prev 잘못
`node.prev.next = node.next` — node.prev 가 None 일 수 있음.

### 함정 4 — Circular 의 끝 조건
While loop — `while cur != head` 등 명시.

### 함정 5 — Memory leak (참조 cycle)
Python의 GC — cycle detection 함. Rust — Rc + Weak 필요.

### 함정 6 — Iterator invalidation
Modification 시 — iterator 깨짐. 별도 변수 보존.

---

## 11. 학습 자료

- CLRS Ch 10
- Java LinkedList docs
- Python deque 구현 (collections module)

---

## 12. 관련

- [[linked-lists]] — Hub
- [[singly-linked-list]] — 비교
- [[lru-cache]] — 응용
- [[../stacks-and-queues/deque]] (TBD)
