---
title: "AVL Tree — 첫 자기 균형 BST"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T16:10:00+09:00
tags:
  - data-structure
  - avl
  - balanced-tree
---

# AVL Tree

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Rotation / Balance factor |

**[[trees|↑ Trees]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

**Adelson-Velsky + Landis (1962)** — 첫 self-balancing BST. 각 노드의 **좌/우 높이 차 ≤ 1**.

---

## 2. Balance Factor

```
BF(node) = height(left) - height(right)
```

### AVL 조건
- 모든 노드 — BF ∈ {-1, 0, 1}

### 위반 시 — rotation

---

## 3. Rotation — 4 가지

### Left Rotation (Right-heavy)
```
   A                B
    \              / \
     B      →     A   C
      \
       C
```

### Right Rotation (Left-heavy)
```
       C            B
      /            / \
     B      →     A   C
    /
   A
```

### Left-Right (LR)
```
       C            C            B
      /            /            / \
     A      →     B      →     A   C
      \          /
       B        A
```

### Right-Left (RL)
- 대칭

---

## 4. Insert

```python
def insert(node, val):
    if not node: return Node(val)
    if val < node.val:
        node.left = insert(node.left, val)
    else:
        node.right = insert(node.right, val)
    
    node.height = 1 + max(height(node.left), height(node.right))
    bf = balance_factor(node)
    
    # 4 cases
    if bf > 1 and val < node.left.val:
        return rotate_right(node)
    if bf < -1 and val > node.right.val:
        return rotate_left(node)
    if bf > 1 and val > node.left.val:
        node.left = rotate_left(node.left)
        return rotate_right(node)
    if bf < -1 and val < node.right.val:
        node.right = rotate_right(node.right)
        return rotate_left(node)
    
    return node
```

---

## 5. Height 보장

### 정리
- N 노드 AVL — 높이 ≤ 1.44 × log₂(N+2) - 1
- 거의 log N (좋은 균형)

### vs Red-Black
- AVL — 더 엄격 (1.44 log N)
- Red-Black — 2 log N
- AVL — 검색 빠름 / Red-Black — insert/delete 빠름 (rotation ↓)

---

## 6. Delete

- BST delete + rotation 으로 균형 회복
- Rotation 횟수 — O(log N) (insert 는 O(1) rotation)

---

## 7. 시간 복잡도

| | 시간 |
| --- | --- |
| Search | O(log N) |
| Insert | O(log N) (rotation O(1)) |
| Delete | O(log N) (rotation up to O(log N)) |
| Min/Max | O(log N) |

---

## 8. AVL vs Red-Black

| | AVL | Red-Black |
| --- | --- | --- |
| **균형** | 엄격 (1.44 log) | 느슨 (2 log) |
| **Search** | 빠름 | 약간 느림 |
| **Insert/Delete** | 약간 느림 | 빠름 |
| **Rotation** | 더 자주 | 적게 |
| **사용** | 검색 빈번 (DB) | 범용 (map/set) |

### 결론
- 모던 stdlib — 대부분 Red-Black (C++ `std::map`, Java `TreeMap`, Linux `rbtree`)
- AVL — 옛 시스템 / 특수

---

## 9. 구현 — 중요 함수

```python
def height(node):
    return node.height if node else 0

def balance_factor(node):
    return height(node.left) - height(node.right)

def rotate_left(z):
    y = z.right
    T2 = y.left
    y.left = z
    z.right = T2
    z.height = 1 + max(height(z.left), height(z.right))
    y.height = 1 + max(height(y.left), height(y.right))
    return y

def rotate_right(z):
    y = z.left
    T3 = y.right
    y.right = z
    z.left = T3
    z.height = 1 + max(height(z.left), height(z.right))
    y.height = 1 + max(height(y.left), height(y.right))
    return y
```

---

## 10. 응용

### Database (옛)
- MS SQL Server — AVL 일부
- 모던 — B-tree

### Memory map (옛)
- 메모리 region 관리

### Routing
- BGP — AVL 일부

### Game development
- 충돌 검출 (interval tree)

---

## 11. 함정

### 함정 1 — Height 업데이트
Rotation 후 — 부모/자식 모두 height 재계산.

### 함정 2 — 4 case 분기
BF 의 부호 + 자식 의 BF 부호 — 4 case.

### 함정 3 — Delete 의 cascading
한 노드 fix → 부모도 부족 → ... → root.

### 함정 4 — Node parent pointer
Rotation 시 — parent 업데이트 필수 (구현 따라).

### 함정 5 — Recursion depth
N = 10^6, 높이 ≈ 20 — 작음. 일반 OK.

---

## 12. 학습 자료

- CLRS Ch 13 (Red-Black 위주)
- "Algorithms" Sedgewick — AVL
- VisuAlgo AVL Tree

---

## 13. 관련

- [[trees]] — Hub
- [[bst]] — 기반
- [[red-black-tree]] — 비교
- [[b-tree]] — 다른 균형
