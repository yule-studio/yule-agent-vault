---
title: "연결 리스트 (Linked Lists)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T06:00:00+09:00
tags:
  - data-structure
  - linked-list
---

# 연결 리스트 (Linked Lists)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 수준 + 11 언어 + 개념 깊이 |

**[[../data-structure|↑ data-structure]]** · **[[../../computer-science|↑↑ computer-science]]**

---

## 1. 한 줄 정의

**각 원소 (노드) 가 다음 원소의 주소를 갖는** 자료구조. 메모리에 흩어진 노드가
포인터 (next) 로 연결되어 논리적인 순서를 형성. 동적 크기 + O(1) 삽입/삭제 (위치
알면) 가 핵심 강점.

---

## 2. 개념의 깊이 — 왜 연결 리스트인가

### 2.1 배열의 한계가 낳은 자료구조

배열은 **연속된 메모리**가 필수 → 두 가지 비용:
1. **크기 변경 = 메모리 재할당 + 복사**.
2. **중간 삽입/삭제 = N 개 shift**.

연결 리스트는 이 두 비용을 회피:
- 각 노드를 **독립된 메모리에 할당**. 크기 제한 없음.
- 노드 간 **포인터로 연결**. 끼워 넣기는 포인터 두 개만 변경 — O(1).

### 2.2 ADT vs 구현

연결 리스트는 **List ADT 의 한 구현**. List ADT 는:
- `add(i, x)`, `remove(i)`, `get(i)`, `set(i, x)`, `size()`.

같은 List ADT 가 배열로 구현되면 → ArrayList / Vec / Python list.
연결 리스트로 구현되면 → LinkedList / std::list / OCaml list.

**ADT 가 같으면 같은 interface**, 구현이 달라도 같은 코드에서 교체 가능.

### 2.3 함수형 언어의 1급 시민

Haskell, OCaml, Lisp, Clojure, Scheme 의 **기본 리스트** 가 연결 리스트:
- `cons` (head + tail) — 1958 Lisp 부터.
- **immutable + persistent** — 새 노드 추가 시 기존 리스트 공유. 매우 효율적.
- 재귀와 자연스럽게 어울림 (`map`, `filter`, `fold`).

```haskell
let list = 1 : 2 : 3 : []        -- [1, 2, 3]
let new_list = 0 : list           -- O(1), list 와 tail 공유
```

### 2.4 메모리 모델의 비용

배열 = **공간 지역성** 좋음 (캐시 친화).
연결 리스트 = **공간 지역성 나쁨** (노드가 흩어짐).

```
배열:   [10][20][30][40]      ← 64 byte 안 (1 캐시 라인)
리스트: [10|*] ─────→ [20|*] ─────→ [30|*]
        0x100         0x800         0x300       ← 흩어짐
```

실측: N=10^6 순회 시 배열이 리스트보다 **10-100배 빠름** (캐시 미스 차이).

**현대 CPU에서는 "이론적 O(1) 삽입" 이 "실제 캐시 미스 비용" 보다 작은 경우가 많아**, 배열 / Dynamic Array 가 거의 항상 더 빠르다. 연결 리스트의 실용성은 줄어듦.

### 2.5 그래도 연결 리스트가 필요한 경우

1. **포인터 안정성** — 노드를 가리키는 포인터/iterator 가 다른 삽입에도 무효화되지 않음. C++ `std::list`.
2. **O(1) 삽입/삭제 (위치 알 때)** — LRU 캐시, OS 의 process list.
3. **함수형 / immutable** — Haskell, Clojure.
4. **합치기 (concat)** — 두 리스트 합치기 O(1) (꼬리에 머리 연결).
5. **메모리가 매우 단편화**되어 큰 연속 블록이 없을 때 (드뭄).

### 2.6 3 가지 변형

| 변형 | 노드 | 사용처 |
| --- | --- | --- |
| **Singly Linked** | next 포인터만 | 가장 단순 + 메모리 최소 |
| **Doubly Linked** | next + prev | 양방향 순회 / O(1) 삭제 / LRU |
| **Circular Linked** | 마지막 노드가 첫 노드 가리킴 | Round-robin 스케줄러 / 게임 턴 |

---

## 3. 메모리 레이아웃 — ASCII 다이어그램

### 단일 연결 (Singly)

```
head
 │
 ▼
┌─────┬─────┐    ┌─────┬─────┐    ┌─────┬──────┐
│ 10  │  *──┼───►│ 20  │  *──┼───►│ 30  │ NULL │
└─────┴─────┘    └─────┴─────┘    └─────┴──────┘
```

각 노드 = `{값, 다음 포인터}`. 마지막 노드의 next = NULL.

### 이중 연결 (Doubly)

```
   ┌──────┬─────┬─────┐    ┌─────┬─────┬─────┐    ┌─────┬─────┬──────┐
   │ NULL │  10 │  *──┼───►│ ◄─* │  20 │  *──┼───►│ ◄─* │  30 │ NULL │
   └──────┴─────┴─────┘    └─────┴─────┴─────┘    └─────┴─────┴──────┘
   ▲                                                                  ▲
   head                                                              tail
```

각 노드 = `{prev, 값, next}`. 양방향 이동 가능 → O(1) 삭제 (이전 노드를 알 필요 없음).

### 환형 연결 (Circular)

```
   ┌─────┬─────┐    ┌─────┬─────┐    ┌─────┬─────┐
   │ 10  │  *──┼───►│ 20  │  *──┼───►│ 30  │  *──┼─┐
   └─────┴─────┘    └─────┴─────┘    └─────┴─────┘ │
   ▲                                                │
   └────────────────────────────────────────────────┘
```

마지막 노드가 head 를 가리킴. 끝없이 순회.

---

## 4. 복잡도 — Array 와 비교

| 연산 | Array | Singly Linked | Doubly Linked |
| --- | --- | --- | --- |
| 인덱스 접근 `[i]` | O(1) | **O(N)** | O(N) |
| 끝에 push | O(1)* | O(N) (꼬리 찾기) / O(1) (tail 포인터 있으면) | O(1) (tail 포인터) |
| 앞에 push | O(N) | **O(1)** | O(1) |
| 중간 삽입 (위치 알 때) | O(N) | **O(1)** | **O(1)** |
| 중간 삭제 (포인터 알 때) | O(N) | O(N) (이전 찾기) | **O(1)** |
| 검색 (값) | O(N) | O(N) | O(N) |
| 공간 | O(N) | O(N) + 포인터 N개 | O(N) + 포인터 2N개 |
| 캐시 효율 | ✅ 매우 좋음 | ❌ 나쁨 | ❌ 나쁨 |

*amortized

---

## 5. 의사 코드

### 단일 연결 — Node 정의 + 삽입

```
struct Node:
    value
    next

function INSERT_HEAD(list, x):
    new_node ← Node(value=x, next=list.head)
    list.head ← new_node

function INSERT_AFTER(node, x):
    new_node ← Node(value=x, next=node.next)
    node.next ← new_node
```

### 단일 연결 — 삭제

```
function DELETE_HEAD(list):
    list.head ← list.head.next

function DELETE_AFTER(node):
    node.next ← node.next.next
```

### 이중 연결 — O(1) 삭제

```
function DELETE_NODE(node):
    node.prev.next ← node.next
    node.next.prev ← node.prev
```

---

## 6. 언어별 구현 (11 언어)

### Python 3 — 직접 구현 (내장 X)

```python
class Node:
    def __init__(self, value, next=None):
        self.value = value
        self.next = next

class LinkedList:
    def __init__(self):
        self.head = None
        self.size = 0

    def add_head(self, x):
        self.head = Node(x, self.head)
        self.size += 1

    def add_tail(self, x):
        if not self.head:
            self.head = Node(x); self.size += 1; return
        cur = self.head
        while cur.next: cur = cur.next
        cur.next = Node(x)
        self.size += 1

    def remove(self, x):
        if not self.head: return
        if self.head.value == x:
            self.head = self.head.next; self.size -= 1; return
        cur = self.head
        while cur.next and cur.next.value != x: cur = cur.next
        if cur.next:
            cur.next = cur.next.next; self.size -= 1

    def __iter__(self):
        cur = self.head
        while cur:
            yield cur.value
            cur = cur.next

# collections.deque — 이중 연결 리스트 (실용)
from collections import deque
dq = deque([1, 2, 3])
dq.appendleft(0)   # O(1)
dq.pop()            # O(1) — 끝
```

### Java

```java
import java.util.LinkedList;
LinkedList<Integer> list = new LinkedList<>();
list.add(1); list.add(2);
list.addFirst(0);          // O(1)
list.removeLast();          // O(1)
list.add(2, 99);            // O(N) — 인덱스 접근 후 삽입

// 직접 구현
class Node {
    int val; Node next;
    Node(int v) { val = v; }
}
class MyLinkedList {
    Node head;
    void addHead(int x) { head = new Node(x); head.next = head; }
}
```

### C++

```cpp
#include <list>
std::list<int> lst = {1, 2, 3};
lst.push_front(0);     // O(1)
lst.push_back(4);       // O(1) (tail 포인터)
lst.erase(it);          // O(1) (iterator 위치)

// forward_list — 단일 연결 (메모리 절약)
#include <forward_list>
std::forward_list<int> fl;
```

### C — 직접 구현

```c
typedef struct Node {
    int val;
    struct Node *next;
} Node;

Node* add_head(Node *head, int x) {
    Node *new_node = malloc(sizeof(Node));
    new_node->val = x;
    new_node->next = head;
    return new_node;   // 새 head 반환
}

Node* find(Node *head, int x) {
    while (head && head->val != x) head = head->next;
    return head;
}

void free_list(Node *head) {
    while (head) {
        Node *tmp = head;
        head = head->next;
        free(tmp);
    }
}
```

### JavaScript

```javascript
// 직접 구현
class Node {
    constructor(val, next = null) { this.val = val; this.next = next; }
}
class LinkedList {
    constructor() { this.head = null; this.size = 0; }
    addHead(x) { this.head = new Node(x, this.head); this.size++; }
    *[Symbol.iterator]() {
        let cur = this.head;
        while (cur) { yield cur.val; cur = cur.next; }
    }
}

// 또는 배열 (대부분의 경우 충분)
const arr = [1, 2, 3];
arr.unshift(0);   // 앞에 O(N)
arr.shift();      // 앞에서 O(N)
```

### TypeScript

```typescript
class Node<T> {
    constructor(public val: T, public next: Node<T> | null = null) {}
}
class LinkedList<T> {
    head: Node<T> | null = null;
    size = 0;
    addHead(x: T): void {
        this.head = new Node(x, this.head);
        this.size++;
    }
}
```

### Kotlin

```kotlin
val list = mutableListOf(1, 2, 3)
val linked = java.util.LinkedList<Int>().apply { addAll(list) }
linked.addFirst(0)
linked.removeLast()

// 직접 구현
class Node(val value: Int, var next: Node? = null)
class LinkedList {
    var head: Node? = null
    fun addHead(x: Int) { head = Node(x, head) }
}
```

### Swift

```swift
// Swift 표준 라이브러리에 LinkedList 없음
class Node<T> {
    var value: T
    var next: Node?
    init(_ value: T, _ next: Node? = nil) {
        self.value = value; self.next = next
    }
}
class LinkedList<T> {
    var head: Node<T>?
    func addHead(_ x: T) { head = Node(x, head) }
}
```

### Go

```go
package main

import "container/list"

func main() {
    l := list.New()
    l.PushBack(1)
    l.PushBack(2)
    l.PushFront(0)
    for e := l.Front(); e != nil; e = e.Next() {
        fmt.Println(e.Value)
    }
}

// 또는 직접
type Node struct {
    val  int
    next *Node
}
func addHead(head *Node, x int) *Node {
    return &Node{val: x, next: head}
}
```

### Rust

```rust
// 표준 std::collections::LinkedList (이중 연결, 잘 안 쓰임)
use std::collections::LinkedList;
let mut list: LinkedList<i32> = LinkedList::new();
list.push_back(1);
list.push_front(0);

// 직접 구현 (소유권 + Box)
struct Node {
    val: i32,
    next: Option<Box<Node>>,
}

impl Node {
    fn new(val: i32) -> Self { Self { val, next: None } }
    fn push_head(self: Option<Box<Self>>, val: i32) -> Option<Box<Self>> {
        Some(Box::new(Node { val, next: self }))
    }
}
```

### C#

```csharp
using System.Collections.Generic;

var list = new LinkedList<int>();
list.AddLast(1); list.AddLast(2);
list.AddFirst(0);
var first = list.First;             // O(1) — LinkedListNode
list.Remove(first);                  // O(1) — node 직접

// 또는 직접
public class Node<T> {
    public T Value;
    public Node<T> Next;
}
```

---

## 7. 변형 / 응용

### (a) LRU 캐시 (Least Recently Used)
이중 연결 리스트 + 해시 맵 — O(1) get / put. [LeetCode 146](https://leetcode.com/problems/lru-cache/).

```python
class LRUCache:
    def __init__(self, capacity):
        from collections import OrderedDict
        self.cache = OrderedDict()
        self.cap = capacity
    
    def get(self, key):
        if key not in self.cache: return -1
        self.cache.move_to_end(key)
        return self.cache[key]
    
    def put(self, key, value):
        if key in self.cache: self.cache.move_to_end(key)
        self.cache[key] = value
        if len(self.cache) > self.cap:
            self.cache.popitem(last=False)
```

### (b) Skip List
연결 리스트 + 다층 인덱스. O(log N) 검색. Redis 의 sorted set 내부.

### (c) XOR Linked List
prev XOR next 하나의 포인터로 두 방향. 메모리 절약 (대신 시작 노드 필요).

### (d) Hash Map 의 Chaining
충돌 시 같은 버킷 안 노드들이 연결 리스트 — 평균 O(1), 최악 O(N).

### (e) Memory Allocator의 Free List
malloc 의 free block들이 연결 리스트로 관리.

### (f) Adjacency List
그래프의 인접 리스트 — 각 노드의 이웃을 연결 리스트로.

---

## 8. 함정 / 안티패턴

### 함정 1 — 인덱스 접근 O(N)
```python
ll[5]   # 연결 리스트는 O(N). 인덱스 자주면 배열 사용.
```

### 함정 2 — Memory leak (C/C++)
`free()` 누락 → 영구 누수. Rust/Go 는 자동 GC, C 는 수동.

### 함정 3 — 사이클 만들기
`A.next = B; B.next = A` → 무한 루프. **Floyd's Cycle Detection** (slow + fast 포인터):

```python
def has_cycle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow == fast: return True
    return False
```

### 함정 4 — Head 변경 시 반환 누락
```python
def add_head(head, x):
    head = Node(x, head)
    # return head 누락 → 호출자 head 안 바뀜
```

### 함정 5 — Rust 의 소유권 / Borrow checker
연결 리스트는 Rust 에서 가장 어려운 자료구조 — `Rc<RefCell<>>` 또는 `unsafe`.

### 함정 6 — Java `LinkedList` 의 `get(i)` 가 O(N)
이름이 List 라고 ArrayList 처럼 쓰면 안 됨. 인덱스 접근 빈번하면 ArrayList.

### 함정 7 — Python 의 deque 가 진짜 연결 리스트?
실제로는 **블록 연결 리스트** (Block-based linked list, 64-element block).
순수 노드 단위가 아님 — 캐시 친화 + O(1) 양 끝.

### 함정 8 — Linked List 가 항상 좋다는 미신
N = 10^6 에서 배열보다 거의 항상 느림. 사용 전에 실측.

---

## 9. 실전 풀이 절차

1. **연결 리스트가 필요한가?** — 거의 모든 코딩테스트는 배열로 가능.
2. **단일 / 이중 / 환형?** — 양방향 / 삭제 필요 → 이중.
3. **dummy head 사용** — `head` 변경 시 코드 단순.
4. **두 포인터 (slow + fast)** — 중간 찾기 / 사이클 / 마지막에서 k번째.
5. **재귀 vs 반복** — 재귀가 깔끔하지만 스택 오버플로 위험.

---

## 10. 대표 문제

### Q1. 연결 리스트 뒤집기 (LeetCode 206)
```python
def reverse(head):
    prev, cur = None, head
    while cur:
        cur.next, prev, cur = prev, cur, cur.next
    return prev
```

### Q2. 두 정렬된 리스트 병합 (LeetCode 21)
```python
def merge(l1, l2):
    dummy = Node(0)
    tail = dummy
    while l1 and l2:
        if l1.val < l2.val: tail.next, l1 = l1, l1.next
        else: tail.next, l2 = l2, l2.next
        tail = tail.next
    tail.next = l1 or l2
    return dummy.next
```

### Q3. 사이클 감지 (LeetCode 141) — Floyd
```python
def has_cycle(head):
    slow = fast = head
    while fast and fast.next:
        slow, fast = slow.next, fast.next.next
        if slow == fast: return True
    return False
```

### Q4. 마지막에서 k번째 노드 (LeetCode 19)
```python
def remove_nth(head, n):
    dummy = Node(0, head)
    slow = fast = dummy
    for _ in range(n + 1): fast = fast.next
    while fast: slow, fast = slow.next, fast.next
    slow.next = slow.next.next
    return dummy.next
```

### Q5. LRU 캐시 (LeetCode 146) — 위 코드 참조

### Q6. 두 수 더하기 (LeetCode 2)
```python
def add(l1, l2):
    dummy = Node(0); tail = dummy; carry = 0
    while l1 or l2 or carry:
        s = (l1.val if l1 else 0) + (l2.val if l2 else 0) + carry
        carry, val = divmod(s, 10)
        tail.next = Node(val); tail = tail.next
        l1 = l1.next if l1 else None
        l2 = l2.next if l2 else None
    return dummy.next
```

---

## 11. 연습 문제

### 입문
- [백준 1158 요세푸스](https://www.acmicpc.net/problem/1158) — 환형
- [LeetCode 206 Reverse Linked List](https://leetcode.com/problems/reverse-linked-list/)
- [LeetCode 21 Merge Two Sorted Lists](https://leetcode.com/problems/merge-two-sorted-lists/)

### 중급
- [LeetCode 146 LRU Cache](https://leetcode.com/problems/lru-cache/)
- [LeetCode 141 Linked List Cycle](https://leetcode.com/problems/linked-list-cycle/)
- [LeetCode 19 Remove Nth From End](https://leetcode.com/problems/remove-nth-node-from-end-of-list/)

### 고급
- [LeetCode 23 Merge K Sorted Lists](https://leetcode.com/problems/merge-k-sorted-lists/) — 힙 결합
- [LeetCode 25 Reverse Nodes in k-Group](https://leetcode.com/problems/reverse-nodes-in-k-group/)
- [LeetCode 460 LFU Cache](https://leetcode.com/problems/lfu-cache/)

---

## 12. 학습 자료

- **CLRS Ch.10** — 표준 학술
- **The Linux Kernel List API** — [`<linux/list.h>`](https://github.com/torvalds/linux/blob/master/include/linux/list.h) — 실무 마스터피스
- **Learn Rust With Entirely Too Many Linked Lists** — [무료](https://rust-unofficial.github.io/too-many-lists/) — Rust 의 깊은 학습
- [VisuAlgo — Linked Lists](https://visualgo.net/en/list)
- "윤성우의 열혈 자료구조" (한국어)

---

## 13. 관련

- [[../arrays-and-strings/arrays-and-strings]] — 배열 vs 리스트 비교
- [[../stacks-and-queues/stacks-and-queues]] — 연결 리스트로 구현
- [[../hash-tables/hash-tables]] — chaining 에 연결 리스트
- [[../../algorithm/dfs-bfs/dfs-bfs]] — 큐에 연결 리스트 사용 가능
- [[../data-structure|↑ data-structure]]
