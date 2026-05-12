---
title: "트리 (Trees)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T07:30:00+09:00
tags:
  - data-structure
  - tree
  - bst
  - red-black-tree
  - b-tree
---

# 트리 (Trees) — BST / AVL / Red-Black / B-Tree

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 수준 + 11 언어 + 균형 트리 깊이 |

**[[../data-structure|↑ data-structure]]** · **[[../../computer-science|↑↑ computer-science]]**

---

## 1. 한 줄 정의

**계층적 (Hierarchical) 자료구조** — 노드들이 부모-자식 관계로 연결.
사이클 없음. 검색·정렬·계층 표현의 토대.

---

## 2. 개념의 깊이

### 2.1 트리 = 계층의 추상

세상의 많은 것이 트리 구조:
- 파일 시스템 (디렉토리 → 파일)
- HTML DOM
- 회사 조직도
- 의사 결정 (decision tree)
- 게임 트리 (체스, 바둑)
- 수학 표현식 `(3 + 4) × 5` → 트리
- 컴파일러 AST (Abstract Syntax Tree)

### 2.2 용어

- **루트 (root)**: 최상위 노드.
- **자식 (child) / 부모 (parent)**: 직접 연결.
- **잎 (leaf)**: 자식 없음.
- **높이 (height)**: 루트에서 가장 깊은 잎까지 거리.
- **깊이 (depth)**: 특정 노드부터 루트까지.
- **차수 (degree)**: 한 노드의 자식 수.

### 2.3 이진 트리 (Binary Tree)

각 노드 ≤ 2 자식. 가장 흔한 트리.

- **완전 이진 트리 (Complete)**: 마지막 레벨 제외 꽉 차고, 마지막은 왼쪽부터.
- **포화 이진 트리 (Full)**: 모든 내부 노드가 정확히 2 자식.
- **균형 이진 트리 (Balanced)**: 좌·우 서브트리 높이 차 ≤ 1.

### 2.4 이진 탐색 트리 (BST)

**왼쪽 < 노드 < 오른쪽** 의 invariant. → 검색 O(log N) 평균.

```
         8
        / \
       3   10
      / \    \
     1   6    14
        / \   /
       4   7 13
```

inorder 순회 (왼쪽 → 노드 → 오른쪽) = 정렬된 순서 `1 3 4 6 7 8 10 13 14`.

**문제**: 정렬된 데이터를 순서대로 삽입하면 한 쪽으로 치우친 트리 → O(N).
→ **균형 트리** 필요.

### 2.5 균형 트리 (Self-Balancing BST)

| 종류 | 발명 | 균형 조건 | 사용 |
| --- | --- | --- | --- |
| **AVL Tree** | Adelson-Velsky & Landis 1962 | 두 서브트리 높이 차 ≤ 1 (엄격) | 검색 빈도 ≫ 삽입 |
| **Red-Black Tree** | Bayer 1972 / Guibas & Sedgewick 1978 | 5 가지 invariant (색) | Java TreeMap, C++ map, Linux 커널 |
| **B-Tree / B+Tree** | Bayer & McCreight 1970 | 각 노드 M 자식 가능 | DB 인덱스 / 파일 시스템 |
| **Treap** | Aragon & Seidel 1989 | BST + 힙 (확률적) | 단순 구현 |
| **Splay Tree** | Sleator & Tarjan 1985 | 자주 접근한 노드 위로 | 캐시 친화 |
| **Skip List** | Pugh 1990 | 트리 아님, 확률적 균형 | Redis sorted set |

### 2.6 AVL vs Red-Black

| 차원 | AVL | Red-Black |
| --- | --- | --- |
| 균형 | 엄격 (높이 차 1) | 느슨 (검은 색만 같음) |
| 검색 | 약간 빠름 (균형 더 좋음) | 보통 |
| 삽입/삭제 | 회전 많음 | 회전 적음 |
| 사용 | 검색 중심 (DB) | 일반 (Java, C++ STL) |

### 2.7 B-Tree / B+Tree — DB 의 핵심

**왜 DB 가 BST 가 아니라 B-Tree?**

- DB 는 디스크 / SSD 에 저장. 디스크 접근 = 100,000 × 메모리.
- 한 페이지 (보통 4-16 KB) 에 **수백 개 키** 저장 가능.
- 한 번의 디스크 IO 로 큰 분기 (M-way) → 트리 높이 매우 낮음 (10억 키도 3-4 단계).

```
B+Tree 예시 (M=4):
                [10 | 20 | 30]
              /     |    |    \
        [1,3,5] [11,13] [22,25] [31,40,50]
        (잎이 양쪽 연결 — 범위 쿼리)
```

**B+Tree** 의 차별점:
- 데이터는 **잎에만**. 내부 노드는 인덱스만.
- 잎이 **연결 리스트** 로 연결 → 범위 쿼리 O(log N + K) 효율.
- PostgreSQL / MySQL / Oracle 의 기본 인덱스.

### 2.8 LSM-Tree — B-Tree 의 대안

쓰기 빈도가 매우 높을 때 (시계열, 로그): B-Tree 의 random IO 부담.

**LSM-Tree (Log-Structured Merge)**:
- 메모리에 정렬된 버퍼 (Memtable).
- 가득 차면 디스크에 sequential write (SSTable).
- 백그라운드 merge.

LevelDB, RocksDB, Cassandra, HBase, ScyllaDB.

---

## 3. 트리 순회 (Traversal)

### Pre-order (전위)
```
순서: 노드 → 왼쪽 → 오른쪽
용도: 트리 복사, 디렉토리 출력
       1
      / \
     2   3       →  1 2 4 5 3
    / \
   4   5
```

### In-order (중위)
```
순서: 왼쪽 → 노드 → 오른쪽
용도: BST 정렬 순서       →  4 2 5 1 3
```

### Post-order (후위)
```
순서: 왼쪽 → 오른쪽 → 노드
용도: 자식 먼저 처리 (트리 삭제, 디렉토리 크기) →  4 5 2 3 1
```

### Level-order (BFS)
```
순서: 깊이별 (큐 사용)     →  1 2 3 4 5
```

---

## 4. 복잡도

| 트리 | Search | Insert | Delete | 최악 |
| --- | --- | --- | --- | --- |
| BST (균형 안 됨) | O(log N) 평균 | O(log N) | O(log N) | O(N) (chain) |
| AVL | O(log N) | O(log N) | O(log N) | O(log N) |
| Red-Black | O(log N) | O(log N) | O(log N) | O(log N) |
| B-Tree (M-way) | O(log_M N) | O(log_M N) | O(log_M N) | O(log_M N) |
| Trie (문자열 K 길이) | O(K) | O(K) | O(K) | O(K) |
| Heap | O(N) | O(log N) | O(log N) (top만) | O(log N) |

---

## 5. 의사 코드

### BST 삽입
```
INSERT(root, x):
    if root == NULL: return Node(x)
    if x < root.value: root.left ← INSERT(root.left, x)
    else if x > root.value: root.right ← INSERT(root.right, x)
    return root
```

### BST 검색
```
SEARCH(root, x):
    if root == NULL: return NULL
    if x == root.value: return root
    if x < root.value: return SEARCH(root.left, x)
    else: return SEARCH(root.right, x)
```

### BST 삭제 (가장 복잡)
```
DELETE(root, x):
    if root == NULL: return NULL
    if x < root.value: root.left ← DELETE(root.left, x)
    elif x > root.value: root.right ← DELETE(root.right, x)
    else:    # found
        if root.left == NULL: return root.right
        if root.right == NULL: return root.left
        # 두 자식: 오른쪽 서브트리의 최솟값으로 대체
        successor ← MIN(root.right)
        root.value ← successor.value
        root.right ← DELETE(root.right, successor.value)
    return root
```

### AVL 회전 (Left rotation)
```
ROTATE_LEFT(node):
    new_root ← node.right
    node.right ← new_root.left
    new_root.left ← node
    update heights
    return new_root
```

---

## 6. 언어별 구현 (11 언어)

### Python 3
```python
# Python 표준 라이브러리에 BST 없음 — sortedcontainers 사용 (라이브러리)
from sortedcontainers import SortedDict, SortedSet, SortedList
sd = SortedDict()
sd['apple'] = 5; sd['banana'] = 3
sd.peekitem(0)   # ('apple', 5) — O(log N)

# 직접 BST
class Node:
    def __init__(self, val):
        self.val = val
        self.left = self.right = None

def insert(root, x):
    if not root: return Node(x)
    if x < root.val: root.left = insert(root.left, x)
    elif x > root.val: root.right = insert(root.right, x)
    return root

def inorder(root):
    if not root: return
    yield from inorder(root.left)
    yield root.val
    yield from inorder(root.right)

# heapq — min-heap (트리의 배열 표현)
import heapq
h = []
heapq.heappush(h, 3); heapq.heappush(h, 1)
heapq.heappop(h)   # 1
```

### Java
```java
import java.util.*;

// TreeMap = Red-Black Tree
TreeMap<String, Integer> tm = new TreeMap<>();
tm.put("apple", 5);
tm.firstKey();       // O(log N)
tm.headMap("banana"); // 범위 쿼리

// TreeSet
TreeSet<Integer> ts = new TreeSet<>();
ts.add(5); ts.add(1); ts.add(3);
ts.floor(4);   // 3 (≤4 중 최대)
ts.ceiling(4); // 5 (≥4 중 최소)

// PriorityQueue = Heap
PriorityQueue<Integer> pq = new PriorityQueue<>();
```

### C++
```cpp
#include <map>
#include <set>
#include <queue>

std::map<std::string, int> m;       // Red-Black Tree
m["apple"] = 5;
m.lower_bound("banana");             // O(log N)

std::set<int> s = {5, 1, 3};
auto it = s.upper_bound(3);          // 5

std::priority_queue<int> pq;         // max-heap
```

### C — 직접 구현
```c
typedef struct Node {
    int val;
    struct Node *left, *right;
} Node;

Node* insert(Node *root, int x) {
    if (!root) {
        Node *n = malloc(sizeof(Node));
        n->val = x; n->left = n->right = NULL;
        return n;
    }
    if (x < root->val) root->left = insert(root->left, x);
    else if (x > root->val) root->right = insert(root->right, x);
    return root;
}

void inorder(Node *root) {
    if (!root) return;
    inorder(root.left);
    printf("%d ", root->val);
    inorder(root.right);
}
```

### JavaScript
```javascript
// JS 표준에 없음. Map (해시 기반) 만.
// npm: red-black-tree, sorted-btree 라이브러리

class Node {
    constructor(val) { this.val = val; this.left = this.right = null; }
}

function insert(root, x) {
    if (!root) return new Node(x);
    if (x < root.val) root.left = insert(root.left, x);
    else if (x > root.val) root.right = insert(root.right, x);
    return root;
}

// 인기 패키지: tinyqueue, js-priority-queue, mnemonist
```

### TypeScript
```typescript
class TreeNode<T> {
    left: TreeNode<T> | null = null;
    right: TreeNode<T> | null = null;
    constructor(public val: T) {}
}

function insert<T>(root: TreeNode<T> | null, x: T): TreeNode<T> {
    if (!root) return new TreeNode(x);
    if (x < root.val) root.left = insert(root.left, x);
    else if (x > root.val) root.right = insert(root.right, x);
    return root;
}
```

### Kotlin
```kotlin
// TreeMap (Java 의 Red-Black)
val tm = java.util.TreeMap<String, Int>()
tm["apple"] = 5
tm.firstKey()

// PriorityQueue
val pq = java.util.PriorityQueue<Int>()

// 직접
class Node(val value: Int, var left: Node? = null, var right: Node? = null)
```

### Swift
```swift
// 표준에 없음. SortedDictionary, BTree 패키지 사용
class Node {
    var val: Int
    var left, right: Node?
    init(_ v: Int) { val = v }
}

func insert(_ root: Node?, _ x: Int) -> Node {
    guard let root = root else { return Node(x) }
    if x < root.val { root.left = insert(root.left, x) }
    else if x > root.val { root.right = insert(root.right, x) }
    return root
}
```

### Go
```go
package main

import "container/heap"

// 표준에 BST 없음 — container/heap (heap만)
// 외부: github.com/google/btree, github.com/petar/GoLLRB

type Node struct {
    Val          int
    Left, Right *Node
}

func insert(root *Node, x int) *Node {
    if root == nil { return &Node{Val: x} }
    if x < root.Val { root.Left = insert(root.Left, x) }
    if x > root.Val { root.Right = insert(root.Right, x) }
    return root
}
```

### Rust
```rust
use std::collections::{BTreeMap, BTreeSet, BinaryHeap};

// BTreeMap = B-Tree (캐시 친화, Red-Black 대안)
let mut m: BTreeMap<&str, i32> = BTreeMap::new();
m.insert("apple", 5);

// BTreeSet
let mut s: BTreeSet<i32> = BTreeSet::new();
s.range(1..10);   // 범위

// 최대 힙
let mut pq: BinaryHeap<i32> = BinaryHeap::new();
pq.push(3); pq.push(1);
pq.pop();   // Some(3)

// 직접 BST
struct Node {
    val: i32,
    left: Option<Box<Node>>,
    right: Option<Box<Node>>,
}
```

### C#
```csharp
using System.Collections.Generic;

// SortedDictionary = Red-Black Tree
var sd = new SortedDictionary<string, int>();
sd["apple"] = 5;

// SortedSet
var ss = new SortedSet<int> { 5, 1, 3 };

// .NET 6+
var pq = new PriorityQueue<string, int>();
pq.Enqueue("apple", 5);
```

---

## 7. 변형 / 응용

### (a) Trie (문자열 트리)
별도 노트 [[../tries/tries]] 에서.

### (b) Segment Tree
구간 합/최댓값 쿼리. 게임 / 통계.

### (c) Fenwick Tree (Binary Indexed Tree)
누적합 변형 + 갱신 O(log N).

### (d) Heap (Min/Max)
별도 노트 [[../heaps/heaps]].

### (e) Suffix Tree / Suffix Array
문자열 매칭 / 게놈 분석.

### (f) Quad Tree / Octree
2D / 3D 공간 분할 (게임 / GIS).

### (g) Decision Tree (ML)
ID3, C4.5, Random Forest, XGBoost.

### (h) Merkle Tree
블록체인, Git, 분산 시스템 무결성 검증.

### (i) Spanning Tree (MST)
[[../../algorithm/graph-theory/graph-theory|graph-theory]] 참조.

---

## 8. 함정 / 안티패턴

### 함정 1 — 정렬된 데이터에 BST 직접 삽입
1, 2, 3, 4 ... 순으로 삽입 → 한 쪽 chain (O(N)). **균형 트리** 또는 셔플.

### 함정 2 — 재귀 깊이 (Python)
Python 기본 1000. N=10^5 BST는 한 쪽 chain 시 스택 오버플로:
```python
import sys
sys.setrecursionlimit(10**6)
```

### 함정 3 — Trie 의 메모리
영어 단어 10만 → 각 노드 26 자식 포인터 → 메모리 폭발. **HashMap 자식** 또는 **DAFSA**.

### 함정 4 — 균형 트리 직접 구현
면접 / 코테에서 AVL/RB 직접 구현 거의 X. 라이브러리 사용.

### 함정 5 — DB 에서 인덱스 남용
인덱스 = 트리 → 쓰기 비용. INSERT 가 빈번한 컬럼에 인덱스는 신중.

### 함정 6 — Heap 의 임의 원소 삭제
`heapq` 는 top 만 O(log N). 임의 원소 삭제는 O(N). **lazy deletion** 패턴.

### 함정 7 — BST 의 삭제 복잡성
두 자식 노드 삭제는 successor 또는 predecessor 사용. 잘못 구현 시 invariant 깨짐.

### 함정 8 — 너무 깊은 재귀를 BFS 로 안 바꿈
DFS 재귀 → 명시적 스택. 큰 트리에서 필수.

---

## 9. 실전 절차

1. **이진 vs 일반?** — 자식 ≤ 2 → 이진.
2. **순서 필요?** — 정렬 → BST / TreeMap. 빈도 → Heap.
3. **균형 보장?** — 자체 구현보다 표준 라이브러리.
4. **순회 종류** — pre/in/post/level. inorder + BST → 정렬.
5. **재귀 vs 반복** — 깊이 크면 명시 스택.

---

## 10. 대표 문제

### Q1. Maximum Depth (LeetCode 104)
```python
def max_depth(root):
    if not root: return 0
    return 1 + max(max_depth(root.left), max_depth(root.right))
```

### Q2. Symmetric Tree (LeetCode 101)
```python
def is_sym(root):
    def mirror(a, b):
        if not a and not b: return True
        if not a or not b: return False
        return a.val == b.val and mirror(a.left, b.right) and mirror(a.right, b.left)
    return mirror(root.left, root.right) if root else True
```

### Q3. Validate BST (LeetCode 98)
```python
def is_bst(root, lo=float('-inf'), hi=float('inf')):
    if not root: return True
    if not lo < root.val < hi: return False
    return is_bst(root.left, lo, root.val) and is_bst(root.right, root.val, hi)
```

### Q4. Lowest Common Ancestor (LeetCode 235 / 236)
BST 일 때:
```python
def lca_bst(root, p, q):
    while root:
        if p.val < root.val and q.val < root.val: root = root.left
        elif p.val > root.val and q.val > root.val: root = root.right
        else: return root
```

### Q5. Level Order Traversal (LeetCode 102)
```python
from collections import deque
def level_order(root):
    if not root: return []
    result = []; q = deque([root])
    while q:
        level = []
        for _ in range(len(q)):
            node = q.popleft(); level.append(node.val)
            if node.left: q.append(node.left)
            if node.right: q.append(node.right)
        result.append(level)
    return result
```

### Q6. Serialize/Deserialize (LeetCode 297)
```python
def serialize(root):
    if not root: return "#"
    return f"{root.val},{serialize(root.left)},{serialize(root.right)}"

def deserialize(data):
    def helper(it):
        x = next(it)
        if x == '#': return None
        node = TreeNode(int(x))
        node.left = helper(it)
        node.right = helper(it)
        return node
    return helper(iter(data.split(',')))
```

---

## 11. 연습 문제

### 입문
- [LeetCode 104 Max Depth](https://leetcode.com/problems/maximum-depth-of-binary-tree/)
- [LeetCode 226 Invert Binary Tree](https://leetcode.com/problems/invert-binary-tree/)
- [LeetCode 100 Same Tree](https://leetcode.com/problems/same-tree/)

### 중급
- [LeetCode 98 Validate BST](https://leetcode.com/problems/validate-binary-search-tree/)
- [LeetCode 102 Level Order](https://leetcode.com/problems/binary-tree-level-order-traversal/)
- [LeetCode 235 LCA in BST](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-search-tree/)
- [LeetCode 105 Build Tree from preorder/inorder](https://leetcode.com/problems/construct-binary-tree-from-preorder-and-inorder-traversal/)

### 고급
- [LeetCode 297 Serialize/Deserialize](https://leetcode.com/problems/serialize-and-deserialize-binary-tree/)
- [LeetCode 124 Maximum Path Sum](https://leetcode.com/problems/binary-tree-maximum-path-sum/)
- [LeetCode 99 Recover BST](https://leetcode.com/problems/recover-binary-search-tree/)
- [백준 1991 트리 순회](https://www.acmicpc.net/problem/1991)

---

## 12. 학습 자료

- **CLRS Ch.12-14** — BST / Red-Black / B-Tree
- **Database Internals (Alex Petrov)** — B-Tree / B+Tree / LSM-Tree
- **Designing Data-Intensive Applications (DDIA) Ch.3** — DB 의 트리
- [VisuAlgo — Trees](https://visualgo.net/en/bst) — BST / AVL / B-Tree 시각화
- [Red-Black Tree 시각화](https://www.cs.usfca.edu/~galles/visualization/RedBlack.html)
- "이것이 자료구조 + 알고리즘이다" (한빛미디어, 직접 구현)

---

## 13. 관련

- [[../heaps/heaps]] — 트리의 배열 구현
- [[../tries/tries]] — 문자열 트리
- [[../graphs/graphs]] — 트리는 사이클 없는 그래프
- [[../../algorithm/dfs-bfs/dfs-bfs]] — 트리 순회 = DFS/BFS
- [[../../database-theory/database-theory]] — B-Tree DB 인덱스 (작성 예정)
- [[../data-structure|↑ data-structure]]
