---
title: "Singly Linked List — 단일 연결 리스트"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T12:00:00+09:00
tags:
  - data-structure
  - linked-list
---

# Singly Linked List

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 구현 / 연산 / 함정 |

**[[linked-lists|↑ Linked Lists]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

**노드가 next 포인터로 연결된** 자료구조. 메모리 비연속. O(1) head 삽입 / 삭제. Array 의 대안.

---

## 2. 구조

```
[1 | →] → [2 | →] → [3 | →] → [4 | NULL]
 head
```

### 노드
```python
class Node:
    def __init__(self, val):
        self.val = val
        self.next = None
```

---

## 3. 연산 — 시간 복잡도

| 연산 | 시간 |
| --- | --- |
| Access by index | O(N) |
| Insert at head | O(1) |
| Insert at tail (tail 포인터 있으면) | O(1) |
| Insert at i | O(N) |
| Delete at head | O(1) |
| Delete by value | O(N) |
| Search | O(N) |
| Reverse | O(N) |

---

## 4. 기본 연산

### Insert at head
```python
def insert_head(head, val):
    node = Node(val)
    node.next = head
    return node    # 새 head
```

### Insert at tail
```python
def insert_tail(head, val):
    node = Node(val)
    if not head: return node
    cur = head
    while cur.next:
        cur = cur.next
    cur.next = node
    return head
```

### Delete value
```python
def delete(head, val):
    dummy = Node(0)
    dummy.next = head
    cur = dummy
    while cur.next:
        if cur.next.val == val:
            cur.next = cur.next.next
            break
        cur = cur.next
    return dummy.next
```

### Reverse
```python
def reverse(head):
    prev = None
    cur = head
    while cur:
        nxt = cur.next
        cur.next = prev
        prev = cur
        cur = nxt
    return prev
```

### Reverse — recursive
```python
def reverse_rec(head):
    if not head or not head.next:
        return head
    new_head = reverse_rec(head.next)
    head.next.next = head
    head.next = None
    return new_head
```

---

## 5. Dummy / Sentinel node

### 정의
- 실제 데이터 없는 노드 — head 앞에
- Edge case (빈 list) 단순화

```python
dummy = Node(0)
dummy.next = head
# ... 작업 ...
return dummy.next
```

### 효과
- Head 변경 시 — return 단순
- if head is None 분기 ↓

---

## 6. 흔한 문제

### 6.1 Find middle
```python
def middle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
    return slow
```

### 6.2 Detect cycle (Floyd's)
```python
def has_cycle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow == fast:
            return True
    return False
```

### 6.3 Remove Nth from end
```python
def remove_nth(head, n):
    dummy = Node(0, head)
    fast = slow = dummy
    for _ in range(n):
        fast = fast.next
    while fast.next:
        slow = slow.next
        fast = fast.next
    slow.next = slow.next.next
    return dummy.next
```

### 6.4 Merge two sorted
```python
def merge(l1, l2):
    dummy = Node(0)
    cur = dummy
    while l1 and l2:
        if l1.val < l2.val:
            cur.next = l1
            l1 = l1.next
        else:
            cur.next = l2
            l2 = l2.next
        cur = cur.next
    cur.next = l1 or l2
    return dummy.next
```

### 6.5 Palindrome check
```python
def is_palindrome(head):
    # 1. Find middle
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
    
    # 2. Reverse second half
    prev = None
    while slow:
        nxt = slow.next
        slow.next = prev
        prev = slow
        slow = nxt
    
    # 3. Compare
    left, right = head, prev
    while right:
        if left.val != right.val:
            return False
        left = left.next
        right = right.next
    return True
```

---

## 7. Array vs Linked List

| | Array | Linked List |
| --- | --- | --- |
| **메모리** | 연속 | 분산 |
| **인덱스 access** | O(1) | O(N) |
| **Insert head** | O(N) | O(1) |
| **Insert tail** | O(1) amortized | O(1) (tail 보유) |
| **Insert middle** | O(N) | O(1) (위치 보유) |
| **Cache locality** | 좋음 | 나쁨 |
| **Memory overhead** | 작음 | 노드 별 pointer (8-16 byte) |
| **Random access** | O(1) | O(N) |

### Bjarne Stroustrup 의 경험적 관찰
- 작은 N (수만) — Array always wins (cache)
- 큰 element / 잦은 insert middle — Linked List 가능 (rare)

---

## 8. 사용 — 실무

### Hash table 의 chaining
- 각 bucket — linked list

### Adjacency list (Graph)
- 노드 별 인접 노드 list

### LRU Cache
- Doubly linked list 활용 (자세히 → [[lru-cache]])

### File system / OS
- Free block list (옛 FAT)
- Process list

---

## 9. 언어별

### C (수동)
```c
typedef struct Node {
    int val;
    struct Node* next;
} Node;

Node* insert(Node* head, int val) {
    Node* node = malloc(sizeof(Node));
    node->val = val;
    node->next = head;
    return node;
}
```

### Python (직접 / collections.deque)
- 직접 구현 위와 같음
- `collections.deque` — doubly linked 비슷한 인터페이스 (실제는 array 기반)

### Java `LinkedList`
- Doubly linked
- `LinkedList<Integer>`
- ArrayList 보다 거의 항상 느림

### Go (container/list)
- doubly linked

### Rust
- `LinkedList<T>` — 거의 안 씀 (cache locality)
- 직접 구현 — ownership 어려움 ("Linked Lists are too hard")

---

## 10. 함정

### 함정 1 — Null pointer
`cur.next.val` — `cur.next` None 일 수 있음.

### 함정 2 — Cycle
무한 loop. 종료 조건 신중.

### 함정 3 — Reverse 의 next 잃기
`cur.next = prev` 전에 — `nxt = cur.next` 보존.

### 함정 4 — Memory leak (C / C++)
free 누락. unique_ptr / shared_ptr.

### 함정 5 — Iteration 중 modification
다음 node — 변경 전 보존.

### 함정 6 — Dummy 잘못 사용
return dummy 대신 dummy.next.

### 함정 7 — 큰 list 의 재귀
Stack overflow. Iterative.

---

## 11. 학습 자료

- CLRS Ch 10
- "Learning to Program with Linked Lists" (CMU)
- "Why Linked Lists Are Underrated" — Bjarne Stroustrup (반박)

---

## 12. 관련

- [[linked-lists]] — Hub
- [[doubly-linked-list]] — 양방향
- [[../arrays-and-strings/dynamic-array]] — 비교
- [[lru-cache]] — 활용
