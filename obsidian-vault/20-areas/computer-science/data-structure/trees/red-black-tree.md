---
title: "Red-Black Tree — 범용 균형 BST"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T16:15:00+09:00
tags:
  - data-structure
  - red-black-tree
  - balanced-tree
---

# Red-Black Tree

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 5 규칙 / rotation / recolor |

**[[trees|↑ Trees]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

**Rudolf Bayer (1972), Guibas+Sedgewick (1978)** — 노드 색깔로 균형 유지. 모던 stdlib 의 default ordered map.

---

## 2. 5 규칙

1. 모든 노드 — 빨강 또는 검정
2. Root — 검정
3. 모든 leaf (NIL) — 검정
4. **빨강의 자식 — 모두 검정** (no two consecutive red)
5. 모든 path (root → NIL leaf) — **같은 수의 검정 노드** ("black height")

### 효과
- 가장 긴 path ≤ 2 × 가장 짧은 path
- 높이 ≤ 2 log₂(N+1)

---

## 3. 시간 복잡도

| | 시간 |
| --- | --- |
| Search | O(log N) |
| Insert | O(log N), rotation ≤ 2 |
| Delete | O(log N), rotation ≤ 3 |

### AVL 대비
- Rotation 적음
- 검색 약간 느림 (균형 느슨)

---

## 4. Insert

### 단계
1. BST insert — 새 노드 — **빨강**
2. 규칙 위반 검사 — 부모 빨강이면 위반 (4)
3. Fixup — rotation / recolor

### Fixup — 6 case (대칭 포함)

#### Uncle 빨강
- Parent + uncle 검정으로
- Grandparent 빨강으로
- Grandparent 재검사 (상승)

#### Uncle 검정
- Rotation + recolor
- 4 sub-case (LL / LR / RR / RL)

---

## 5. Delete

### 단계
1. BST delete
2. 검정 노드 삭제 → "extra black"
3. Fixup — 4 case

### 함정
- 매우 복잡 — 책 페이지 만큼
- 보통 — library 사용

---

## 6. Implementation 의 트릭

### Sentinel NIL
- 모든 leaf = 한 sentinel NIL 노드 (color 검정)
- Edge case 단순화

### Parent pointer
- Rotation / fixup 에 필수
- 메모리 ↑

---

## 7. 사용 — 모던 stdlib

### C++ `std::map` / `std::set` / `std::multimap`
- Red-Black Tree (대부분 구현)

### Java `TreeMap` / `TreeSet`
- Red-Black Tree

### Linux kernel `rbtree`
- include/linux/rbtree.h
- 매우 많은 곳 (vma, ext4, ...)

### Java `HashMap` (Java 8+) treeify
- Chain 8 이상 — Red-Black Tree

### CFS scheduler
- Linux 의 process scheduler — RB tree

자세히 → operating-system/scheduling

---

## 8. AVL vs Red-Black 의 실측

### AVL
- 검색 5-10% 빠름
- Insert/Delete 약간 느림

### Red-Black
- Insert/Delete 빠름
- 일반 워크로드 — 약간 빠름

### 선택
- 검색 빈번 (DB 의 옛 일부) — AVL
- 모던 / 범용 — Red-Black

---

## 9. Treap

### 정의
- BST + Heap 결합
- Priority — random
- BST 순서 유지 + heap 순서 (priority 로) 유지

### 효과
- Expected O(log N) — 균형 자동
- 구현 단순 (rotation 만)

### 사용
- 일부 경쟁 알고리즘
- 외부 메모리

자세히 → competitive-programming

---

## 10. Splay Tree

### 정의
- Access 시 — 노드를 root 로 (rotation)
- 자주 접근 — 빠름 (cache-like)

### 효과
- Amortized O(log N)
- Worst case — O(N) (가능)

### 사용
- LRU 같은 사용
- 일부 컴파일러

---

## 11. WAVL / Scapegoat Tree

### WAVL — Weak AVL
- AVL + Red-Black 의 hybrid
- 모던 (Haeupler 2015)

### Scapegoat
- α-weight balance
- Rebuild on imbalance

---

## 12. 구현 — Red-Black tree (간단화)

### 노드
```python
class Node:
    def __init__(self, val, color='red'):
        self.val = val
        self.left = None
        self.right = None
        self.parent = None
        self.color = color    # 'red' or 'black'
```

### 직접 구현 — 매우 복잡
- 보통 — library / 표준 사용
- LeetCode 등 — 보통 출제 X

---

## 13. 함정

### 함정 1 — 직접 구현 어려움
실수 많음. 표준 라이브러리 권장.

### 함정 2 — Memory 의 color bit
1 bit 만 필요. 보통 — int 1 byte.

### 함정 3 — Persistent 의 부담
함수형 Red-Black — Functional Red-Black Trees (Okasaki)

### 함정 4 — 균형 → Worst case 약속
Average — 좋음. Worst — 2 log N.

### 함정 5 — Iterator 의 next
Next — in-order successor. O(amortized 1).

---

## 14. 학습 자료

- CLRS Ch 13 — 가장 자세
- "Algorithms" Sedgewick — Left-Leaning Red-Black (단순화)
- "Purely Functional Data Structures" (Okasaki) — 함수형
- Linux source — rbtree.c

---

## 15. 관련

- [[trees]] — Hub
- [[bst]] / [[avl-tree]] — 비교
- [[b-tree]] — 다른 balance
- [[../hash-tables/collision-resolution]] — Java treeify
