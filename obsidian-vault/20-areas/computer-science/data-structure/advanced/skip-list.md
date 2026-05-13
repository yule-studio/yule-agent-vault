---
title: "Skip List — 확률적 정렬 자료구조"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:15:00+09:00
tags:
  - data-structure
  - skip-list
  - probabilistic
---

# Skip List

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 다중 레벨 / 확률 / Redis |

**[[advanced|↑ Advanced]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

**여러 레벨의 정렬 linked list** — O(log N) 검색 / 삽입 / 삭제 (확률적). 1989 William Pugh. Tree 의 대안.

---

## 2. 구조

```
Level 3:  HEAD ────────────────────────→ TAIL
                                  
Level 2:  HEAD ──────→ 17 ────────→ 25 → TAIL
                
Level 1:  HEAD → 6 → 17 → 21 → 25 → 30 → TAIL

Level 0:  HEAD → 6 → 9 → 17 → 21 → 25 → 30 → TAIL
                                  
모든 레벨 — 정렬 linked list
높은 레벨 — 더 적은 노드 (빠른 검색)
```

### 노드 — 자식 노드의 forward 포인터 array

---

## 3. 동작

### Search
1. Top level — 가장 큰 < target 까지
2. 다음 level 내려가서 — 계속
3. Bottom level 도달

```python
def search(head, target):
    cur = head
    for level in range(max_level, -1, -1):
        while cur.forward[level] and cur.forward[level].val < target:
            cur = cur.forward[level]
    cur = cur.forward[0]
    return cur if cur and cur.val == target else None
```

### Insert
1. 검색 path 저장 (각 level 의 직전 노드)
2. Random level 생성 (coin flip)
3. 각 level — 노드 삽입

```python
def random_level():
    level = 0
    while random.random() < 0.5 and level < max_level:
        level += 1
    return level

def insert(head, val):
    update = [None] * (max_level + 1)
    cur = head
    for level in range(max_level, -1, -1):
        while cur.forward[level] and cur.forward[level].val < val:
            cur = cur.forward[level]
        update[level] = cur
    
    new_level = random_level()
    new_node = Node(val, new_level + 1)
    for level in range(new_level + 1):
        new_node.forward[level] = update[level].forward[level]
        update[level].forward[level] = new_node
```

---

## 4. 시간 — 확률적

| | 평균 | 최악 |
| --- | --- | --- |
| Search | O(log N) | O(N) |
| Insert | O(log N) | O(N) |
| Delete | O(log N) | O(N) |

### 평균 분석
- 각 노드 — level k 확률 = 1/2^k
- 평균 검색 — log N 단계

### 최악 — 매우 드뭄
- 모든 노드 — level 0 (확률 매우 낮음)

---

## 5. Skip List vs Balanced BST

| | Skip List | BBST (AVL/RB) |
| --- | --- | --- |
| **구조** | linked list 다중 | tree |
| **시간** | E[O(log N)] | O(log N) 보장 |
| **구현** | 단순 | 복잡 |
| **Concurrent** | 쉬움 | 어려움 |
| **메모리** | 약간 ↑ | 알맞 |
| **Cache** | 보통 | 보통 |

### 선택
- 구현 단순 / 동시성 — Skip List
- Strict O(log N) 보장 — BBST

---

## 6. 응용

### Redis Sorted Set (ZSET)
- Skip List + Hash
- Score 기반 정렬
- O(log N) ZRANGE, ZADD

### LevelDB / RocksDB — Memtable
- 정렬 + 빠른 insert
- Skip List 기반

### Java `ConcurrentSkipListMap`
- 동시성 친화 TreeMap

### 미디어 — 시간 순 인덱스
- 큰 timeline + 임의 위치 검색

---

## 7. 확률 의 의미

### 같은 N — 다른 Skip List
- Random level → 매 build 다른 구조
- 평균 성능 동일

### Worst case 회피
- Adversarial input X (random level)
- BST 의 sorted insert 같은 함정 X

---

## 8. 변형

### Deterministic Skip List
- 1-2-3 skip list — 같은 시간 보장
- Random 없음
- 구현 더 복잡

### Indexable Skip List
- 각 노드 — span (다음까지 길이)
- O(log N) k-th element

---

## 9. Concurrent Skip List

### Lock-free
- CAS 기반
- Java `ConcurrentSkipListMap` — Doug Lea

### 효과
- Reader / writer 동시
- BBST 의 lock 복잡성 회피

---

## 10. 구현

```python
import random

class Node:
    def __init__(self, val, level):
        self.val = val
        self.forward = [None] * (level + 1)

class SkipList:
    MAX_LEVEL = 16
    P = 0.5
    
    def __init__(self):
        self.head = Node(None, self.MAX_LEVEL)
        self.level = 0
    
    def random_level(self):
        lvl = 0
        while random.random() < self.P and lvl < self.MAX_LEVEL:
            lvl += 1
        return lvl
    
    def search(self, target):
        cur = self.head
        for i in range(self.level, -1, -1):
            while cur.forward[i] and cur.forward[i].val < target:
                cur = cur.forward[i]
        cur = cur.forward[0]
        return cur if cur and cur.val == target else None
    
    def insert(self, val):
        update = [None] * (self.MAX_LEVEL + 1)
        cur = self.head
        for i in range(self.level, -1, -1):
            while cur.forward[i] and cur.forward[i].val < val:
                cur = cur.forward[i]
            update[i] = cur
        
        new_level = self.random_level()
        if new_level > self.level:
            for i in range(self.level + 1, new_level + 1):
                update[i] = self.head
            self.level = new_level
        
        new_node = Node(val, new_level)
        for i in range(new_level + 1):
            new_node.forward[i] = update[i].forward[i]
            update[i].forward[i] = new_node
```

---

## 11. 함정

### 함정 1 — Worst case
거의 0 확률이지만 — 이론상 O(N).

### 함정 2 — Random source
predictable random (PRNG) — attacker 가 worst case 강요.
Cryptographic random / private seed.

### 함정 3 — Level 의 max
log N 정도. 너무 작음 — 성능 ↓. 16-32 일반.

### 함정 4 — Concurrent 의 deletion
Lock-free deletion — 미묘. Java 구현 참고.

### 함정 5 — Memory
각 노드 — 다중 forward. BBST 와 비슷.

---

## 12. 학습 자료

- "Skip Lists: A Probabilistic Alternative to Balanced Trees" — Pugh 1990
- Redis source — t_zset.c
- Java `ConcurrentSkipListMap` source
- CLRS 가 다루지 않음 (Sedgewick 다룸)

---

## 13. 관련

- [[advanced]] — Hub
- [[../trees/red-black-tree]] — 비교
- [[../linked-lists/doubly-linked-list]] — 기반
- Redis sorted set
