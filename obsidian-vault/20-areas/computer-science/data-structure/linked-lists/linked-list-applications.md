---
title: "Linked List 응용 — 실무 사례"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T12:15:00+09:00
tags:
  - data-structure
  - linked-list
  - applications
---

# Linked List 응용 — 실무 사례

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | hash chain / adj list / OS / git |

**[[linked-lists|↑ Linked Lists]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 정적 array vs linked list — 현실

### Stroustrup 의 관찰
- 작은 N — Array 가 항상 빠름 (cache locality)
- Linked List — **특정 use case** 에서만

### 그래도 — 어디 쓰는가?
- 노드 자체 가지고 있을 때 — O(1) 삭제
- 큰 단위의 chunk 들 (Python `deque`)
- 메모리 제약 — chunk allocation
- 임베디드 — fixed-size pool

---

## 2. Hash Table — Chaining

### 동작
- 각 bucket 의 collision — linked list
- 같은 hash 의 entry 들

```
hash table:
  [0] → [k1:v1] → [k4:v4]
  [1] → NULL
  [2] → [k2:v2] → [k7:v7] → [k9:v9]
  [3] → [k3:v3]
```

### 효과
- 충돌 — chain 길이 1-2 (resize 잘 되면)
- 평균 O(1)
- 최악 (한 bucket 으로 몰리면) O(N)

### Java HashMap (~8 entries 이상 — Red-Black Tree)
- Treeify 임계 — chain 너무 길면 tree
- 최악 O(log N)

자세히 → [[../hash-tables/separate-chaining]] (TBD)

---

## 3. Adjacency List (Graph)

### 정의
- 각 노드 별 — 인접 노드의 linked list

```
1: → 2 → 3
2: → 1 → 4
3: → 1
4: → 2
```

### 효과
- Sparse graph — 메모리 절약
- BFS / DFS — O(V + E)

### 대안
- Adjacency matrix — O(V²) 메모리, O(1) edge check
- 모던 — `Vec<Vec<u32>>` (Rust) — 실은 array of arrays

자세히 → [[../graphs/graph-representation]] (TBD)

---

## 4. Implementation of Stack / Queue

### Stack — top 에서 push/pop
```python
class StackLL:
    def __init__(self):
        self.head = None
    
    def push(self, val):
        node = Node(val)
        node.next = self.head
        self.head = node
    
    def pop(self):
        if not self.head: return None
        val = self.head.val
        self.head = self.head.next
        return val
```

### Queue — singly + tail
```python
class QueueLL:
    def __init__(self):
        self.head = None
        self.tail = None
    
    def enqueue(self, val):
        node = Node(val)
        if not self.head:
            self.head = self.tail = node
        else:
            self.tail.next = node
            self.tail = node
    
    def dequeue(self):
        if not self.head: return None
        val = self.head.val
        self.head = self.head.next
        if not self.head: self.tail = None
        return val
```

→ Array 기반 — circular buffer 가 보통 더 빠름.

---

## 5. OS — Process / Task list

### Linux `task_struct`
- doubly linked list 의 모든 process
- `list_for_each_entry(p, &init_task->children, sibling)`

### 효과
- O(1) 노드 삭제 (kill)
- 순회 — 모든 process

### 매크로
```c
#define list_for_each_entry(pos, head, member) ...
```

---

## 6. Memory allocator — Free list

### Free block 의 linked list
```
free_list → [block1] → [block5] → [block9] → NULL
```

- malloc — head 에서 take
- free — head 에 push

### 효과
- Allocation O(1)
- Fragmentation — best-fit / first-fit 따라

### Implementation
- Glibc malloc — small/large bin (free lists)
- jemalloc — size class 별
- TCMalloc (Google)

---

## 7. Git — Object DAG

### Git commit chain
```
commit C ─── parent ──> commit B ─── parent ──> commit A
```

- 각 commit — parent reference (한 노드 가리키는 linked list-like)
- Merge — 2 parent (DAG)

### 효과
- History 탐색 — git log
- Branch — 다른 chain
- Reset — chain 의 옛 노드로

---

## 8. Polynomial 표현

### Sparse polynomial
```
3x^100 + 5x^50 + 7
```

### Linked list
```
[coeff:3, exp:100] → [coeff:5, exp:50] → [coeff:7, exp:0]
```

### 효과
- Sparse — 메모리 절약
- Multiply / add — linked list 순회

---

## 9. Undo / Redo

### Doubly linked list of states
```
[state1] ⇄ [state2] ⇄ [state3] (현재)
```

### Undo
- prev 로 이동

### Redo
- next 로 이동

### 새 action — 현재 이후 chain 제거 (branch X)

---

## 10. Event scheduling (Calendar)

### 시간 순 linked list
```
[2pm meeting] → [3pm call] → [5pm dinner]
```

- Insert — 시간 따라 위치
- O(N) insert / O(1) earliest

### Priority Queue (heap) 가 더 효율

---

## 11. File System

### Old FAT (File Allocation Table)
- File 의 block 들 — linked list
- Block A → Block X → Block Y → ... → EOF

### 한계
- Random access O(N)
- 모던 (ext4, NTFS) — 다른 구조 (B+tree, extent)

---

## 12. Web browser — Tab history

### Doubly linked list
- 각 tab — back / forward
- 자세히 → [[lru-cache]] (cache 와 다름, history 는 array 도 흔함)

---

## 13. Language internals

### Python `collections.deque`
- 사실은 **block 의 doubly linked list**
- 각 block — 약 64 원소의 array
- O(1) 양 끝 + 적당한 cache

### Java `LinkedList`
- 진짜 doubly linked
- 거의 안 쓰임 (ArrayList 권장)

### C++ `std::list`
- 진짜 doubly linked
- `std::forward_list` — singly

### Go `container/list`
- doubly linked

---

## 14. 학습 자료

- "Why You Should Avoid Linked Lists" — Stroustrup (반대 의견)
- Linux kernel `list.h`
- "Modern Operating Systems" (Tanenbaum) — process list

---

## 15. 관련

- [[linked-lists]] — Hub
- [[singly-linked-list]] / [[doubly-linked-list]]
- [[../hash-tables/hash-tables]] — Chaining
- [[../graphs/graphs]] — Adjacency list
