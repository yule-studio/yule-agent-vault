---
title: "BST — Binary Search Tree"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T16:05:00+09:00
tags:
  - data-structure
  - bst
  - tree
---

# BST — Binary Search Tree

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Insert / Delete / 균형 문제 |

**[[trees|↑ Trees]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

각 노드의 **왼쪽 < 자기 < 오른쪽** 인 binary tree. 정렬된 검색 / 삽입 / 삭제 O(log N) (균형 시).

---

## 2. BST property

```
       5
      / \
     3   8
    / \ / \
   1  4 7  9

모든 노드:
    left subtree 의 모든 값 < node.val < right subtree 의 모든 값
```

### In-order traversal — sorted
- 1, 3, 4, 5, 7, 8, 9

---

## 3. 시간 복잡도

| | Balanced | Worst (skewed) |
| --- | --- | --- |
| Search | O(log N) | O(N) |
| Insert | O(log N) | O(N) |
| Delete | O(log N) | O(N) |
| Min/Max | O(log N) | O(N) |
| In-order | O(N) | O(N) |

### 균형 보장 X
- 정렬 순서 insertion → skewed → O(N)
- → AVL / Red-Black 등 self-balancing 필요

---

## 4. 연산

### Search
```python
def search(root, target):
    if not root or root.val == target:
        return root
    if target < root.val:
        return search(root.left, target)
    return search(root.right, target)
```

### Insert
```python
def insert(root, val):
    if not root:
        return Node(val)
    if val < root.val:
        root.left = insert(root.left, val)
    elif val > root.val:
        root.right = insert(root.right, val)
    return root
```

### Delete — 3 경우
1. Leaf — 그냥 제거
2. 자식 1 — 자식 으로 대체
3. 자식 2 — successor (right subtree 의 min) 또는 predecessor 와 swap → 삭제

```python
def delete(root, val):
    if not root: return None
    if val < root.val:
        root.left = delete(root.left, val)
    elif val > root.val:
        root.right = delete(root.right, val)
    else:
        # case 1: no children
        if not root.left and not root.right:
            return None
        # case 2: one child
        if not root.left: return root.right
        if not root.right: return root.left
        # case 3: two children
        succ = root.right
        while succ.left:
            succ = succ.left
        root.val = succ.val
        root.right = delete(root.right, succ.val)
    return root
```

---

## 5. Successor / Predecessor

### Inorder Successor
- right subtree 있음 — right subtree 의 min
- 없음 — 부모 따라 올라가 (왼쪽 자식인 부모 찾기)

```python
def successor(root, target):
    succ = None
    while root:
        if target < root.val:
            succ = root
            root = root.left
        else:
            root = root.right
    return succ
```

---

## 6. Validate BST

### 잘못된 방법
```python
def is_bst_wrong(root):
    if not root: return True
    if root.left and root.left.val >= root.val: return False
    if root.right and root.right.val <= root.val: return False
    return is_bst_wrong(root.left) and is_bst_wrong(root.right)
```

### 함정 — 멀리 있는 노드 비교 안 함
```
    5
   / \
  3   8
 / \
1  6     ← BST 위반! (6 > 5)
```

### 올바른 — min/max bound
```python
def is_bst(root, low=float('-inf'), high=float('inf')):
    if not root: return True
    if not (low < root.val < high):
        return False
    return (is_bst(root.left, low, root.val) and
            is_bst(root.right, root.val, high))
```

### In-order 활용
- in-order traversal — sorted 여야

---

## 7. K-th smallest

```python
def kth_smallest(root, k):
    stack = []
    cur = root
    while cur or stack:
        while cur:
            stack.append(cur)
            cur = cur.left
        cur = stack.pop()
        k -= 1
        if k == 0:
            return cur.val
        cur = cur.right
```

### Order statistics tree
- 각 노드 — subtree size 저장
- O(log N) k-th query

---

## 8. Range Query

### [L, R] 범위 합 / 노드
```python
def range_sum(root, L, R):
    if not root: return 0
    if root.val < L:
        return range_sum(root.right, L, R)
    if root.val > R:
        return range_sum(root.left, L, R)
    return (root.val
            + range_sum(root.left, L, R)
            + range_sum(root.right, L, R))
```

---

## 9. Skewed — 문제

### 정렬 데이터 insert
```python
bst.insert(1)
bst.insert(2)
bst.insert(3)
bst.insert(4)
# 결과: linked list (height N)
```

### 결과
- O(N) — 매 연산
- Linked list 와 다를 바 없음

### 해결 — Self-balancing
- AVL — 자세히 [[avl-tree]]
- Red-Black — 자세히 [[red-black-tree]]
- Treap, Splay, B-tree

---

## 10. C++ `std::map`, Java `TreeMap`

### C++ `std::map` / `std::set`
- Red-Black Tree
- Ordered iteration
- O(log N) ops

```cpp
std::map<int, std::string> m;
m[5] = "five";
m.lower_bound(3);   // 3 이상 첫 key
m.upper_bound(7);   // 7 초과 첫 key
```

### Java `TreeMap` / `TreeSet`
- Red-Black Tree
- `ceilingKey`, `floorKey`, `higherKey`, `lowerKey`

```java
TreeMap<Integer, String> m = new TreeMap<>();
m.put(5, "five");
m.ceilingKey(3);    // 3 이상 첫 key
```

### Python — 표준 없음
- `sortedcontainers` (3rd party)
- 또는 `bisect` + list

---

## 11. BBST 의 종류

| | 균형 | Insert | Delete | 사용 |
| --- | --- | --- | --- | --- |
| AVL | 매우 엄격 | O(log N) | O(log N) | 검색 빈번 |
| Red-Black | 느슨 | O(log N) | O(log N) | 범용 (map / set) |
| B-Tree | 노드 별 많은 키 | O(log N) | O(log N) | DB index |
| Splay | 일부 자주 — top | amortized | amortized | Cache-like |
| Treap | random | E[O(log N)] | E[O(log N)] | 구현 단순 |

자세히 → [[avl-tree]] / [[red-black-tree]] / [[b-tree]]

---

## 12. 함정

### 함정 1 — Skewed insertion
정렬 데이터 → linked list. Self-balancing.

### 함정 2 — 중복 key
일반 BST — 중복 정의 X. Left 또는 right 일관성.

### 함정 3 — Float 비교
부동소수 ===. Epsilon 비교.

### 함정 4 — Delete 의 successor swap
val 만 복사. 노드 자체 swap 시 — parent 포인터 등.

### 함정 5 — Recursion depth
큰 N + skewed — stack overflow. 균형 트리 또는 iterative.

### 함정 6 — Validate 의 single-node check
바로 자식만 비교 — 멀리 있는 노드 위반 못 잡음.

---

## 13. 학습 자료

- CLRS Ch 12
- "Algorithms" Sedgewick — BST
- VisuAlgo BST visualization

---

## 14. 관련

- [[trees]] — Hub
- [[binary-tree]] — 기반
- [[avl-tree]] / [[red-black-tree]] — 균형
- [[b-tree]] — DB index
