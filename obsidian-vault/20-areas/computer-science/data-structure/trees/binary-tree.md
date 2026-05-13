---
title: "Binary Tree — 이진 트리 / 순회"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T16:00:00+09:00
tags:
  - data-structure
  - tree
  - traversal
---

# Binary Tree — 이진 트리 / 순회

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 종류 / 순회 (pre/in/post/level) |

**[[trees|↑ Trees]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

각 노드가 **최대 2 개의 자식** 을 가지는 트리. BST / 표현식 트리 / heap / 압축 트리의 기반.

---

## 2. 종류

| 종류 | 정의 |
| --- | --- |
| **Full** | 모든 노드 — 0 또는 2 자식 |
| **Complete** | 마지막 레벨 제외 — full, 마지막 좌측부터 |
| **Perfect** | 모든 leaf 같은 깊이 + full |
| **Balanced** | 좌/우 높이 차 작음 (정의 다양) |
| **Skewed** | 한쪽으로만 (degenerate — linked list) |

### Heap = Complete binary tree
- Array 표현 자연스러움
- 자세히 → [[../heaps/binary-heap]]

---

## 3. 표현

### 노드 기반
```python
class Node:
    def __init__(self, val):
        self.val = val
        self.left = None
        self.right = None
```

### Array 기반 (complete tree 만)
```
       1
      / \
     2   3
    / \ / \
   4  5 6  7

arr: [_, 1, 2, 3, 4, 5, 6, 7]  (1-base)
parent(i) = i / 2
left(i) = 2i
right(i) = 2i + 1
```

---

## 4. 순회 — DFS

### Pre-order (Root → Left → Right)
```python
def pre(node):
    if not node: return
    print(node.val)
    pre(node.left)
    pre(node.right)
```

### In-order (Left → Root → Right)
```python
def inorder(node):
    if not node: return
    inorder(node.left)
    print(node.val)
    inorder(node.right)
```

### Post-order (Left → Right → Root)
```python
def post(node):
    if not node: return
    post(node.left)
    post(node.right)
    print(node.val)
```

### BST 의 inorder — sorted
- BST 의 inorder traversal — 정렬된 순서

---

## 5. 순회 — BFS / Level-order

```python
from collections import deque

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

---

## 6. Iterative 순회

### Pre-order — stack
```python
def pre_iter(root):
    if not root: return
    stack = [root]
    while stack:
        node = stack.pop()
        print(node.val)
        if node.right: stack.append(node.right)
        if node.left: stack.append(node.left)
```

### In-order — stack
```python
def inorder_iter(root):
    stack = []
    cur = root
    while cur or stack:
        while cur:
            stack.append(cur)
            cur = cur.left
        cur = stack.pop()
        print(cur.val)
        cur = cur.right
```

### Post-order — 두 stack 또는 modified pre
```python
def post_iter(root):
    if not root: return
    stack1, stack2 = [root], []
    while stack1:
        node = stack1.pop()
        stack2.append(node)
        if node.left: stack1.append(node.left)
        if node.right: stack1.append(node.right)
    while stack2:
        print(stack2.pop().val)
```

---

## 7. Morris Traversal — O(1) 공간

### 정의
- Threaded binary tree 활용
- 임시 — leaf 의 right 를 다음 노드로
- 끝나면 복원

```python
def morris_inorder(root):
    cur = root
    while cur:
        if not cur.left:
            print(cur.val)
            cur = cur.right
        else:
            # find predecessor
            pred = cur.left
            while pred.right and pred.right != cur:
                pred = pred.right
            if not pred.right:
                pred.right = cur
                cur = cur.left
            else:
                pred.right = None
                print(cur.val)
                cur = cur.right
```

### 효과
- O(N) 시간, O(1) 공간 (recursion / stack X)

### 단점
- 트리 임시 변경 → thread-unsafe

---

## 8. Properties

### 높이 / 깊이
- **Depth** — root 에서 노드까지 거리
- **Height** — 노드에서 가장 먼 leaf 까지

### Balanced 의 정의 (다양)
- AVL — 좌/우 height 차 ≤ 1
- Red-Black — 높이 ≤ 2 log N
- Weight-balanced — node 수 ≤ 2배

### 노드 수와 높이
- Perfect tree of height h: 2^(h+1) - 1 노드
- N 노드 — height ≥ log₂(N+1) - 1

---

## 9. Diameter

### 정의
- 트리에서 가장 긴 path (노드 수)
- 두 leaf 의 경로 중 가장 김

```python
def diameter(root):
    self.ans = 0
    def height(node):
        if not node: return 0
        l = height(node.left)
        r = height(node.right)
        self.ans = max(self.ans, l + r)
        return 1 + max(l, r)
    height(root)
    return self.ans
```

---

## 10. Lowest Common Ancestor (LCA)

### Recursive
```python
def lca(root, p, q):
    if not root or root == p or root == q:
        return root
    left = lca(root.left, p, q)
    right = lca(root.right, p, q)
    if left and right:
        return root
    return left or right
```

### BST 에서는 — O(log N)
```python
def lca_bst(root, p, q):
    while root:
        if p.val < root.val and q.val < root.val:
            root = root.left
        elif p.val > root.val and q.val > root.val:
            root = root.right
        else:
            return root
```

---

## 11. Serialize / Deserialize

### Pre-order + null marker
```python
def serialize(root):
    result = []
    def dfs(node):
        if not node:
            result.append("null")
            return
        result.append(str(node.val))
        dfs(node.left)
        dfs(node.right)
    dfs(root)
    return ",".join(result)

def deserialize(data):
    vals = iter(data.split(","))
    def build():
        v = next(vals)
        if v == "null": return None
        node = Node(int(v))
        node.left = build()
        node.right = build()
        return node
    return build()
```

---

## 12. 응용

### 표현식 트리
```
    (+)
   /   \
  (*)   (-)
  / \   / \
 2   3 4   5

(2 * 3) + (4 - 5)
```

### Decision Tree (ML)
- 분기 — feature 비교

### Game tree (AI)
- Minimax / Alpha-beta pruning

### Huffman tree
- 자세히 → [[../heaps/heap-applications]]

### Parse tree (컴파일러)
- AST (Abstract Syntax Tree)

---

## 13. 함정

### 함정 1 — Recursion depth
큰 skewed tree — stack overflow. Iterative + explicit stack.

### 함정 2 — Null check
`node.left.val` — `node.left` None 일 수 있음.

### 함정 3 — Sibling 접근
부모 정보 없으면 sibling 접근 불가. 부모 포인터 또는 BFS.

### 함정 4 — Modification 중 순회
순회 중 트리 변경 — 정의되지 않음. Snapshot.

### 함정 5 — Heap 의 binary tree 의미
Heap — complete tree (특수). 일반 binary tree 와 다름.

### 함정 6 — In-order 의 BST 가정
일반 binary tree — in-order 정렬 X (BST 만).

---

## 14. 학습 자료

- CLRS Ch 12
- LeetCode Tree tag
- "Trees and Hierarchical Data" — DDIA

---

## 15. 관련

- [[trees]] — Hub
- [[bst]] — Binary Search Tree
- [[../heaps/binary-heap]] — Complete tree
- [[../../algorithm/algorithm]] — DFS / BFS
