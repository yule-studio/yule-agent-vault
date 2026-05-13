---
title: "LRU Cache — Least Recently Used"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T12:10:00+09:00
tags:
  - data-structure
  - cache
  - lru
---

# LRU Cache — Least Recently Used

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Hash + DLL / 변형 |

**[[linked-lists|↑ Linked Lists]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

**가장 오래된 접근 항목** 부터 evict 하는 cache. Hash Map + Doubly Linked List 결합으로 O(1) get / put.

---

## 2. 동작

```
Capacity 3:

put(1, A)     → [1:A]                        head
put(2, B)     → [2:B] ⇄ [1:A]                head ← MRU
put(3, C)     → [3:C] ⇄ [2:B] ⇄ [1:A]        tail ← LRU

get(1)        → A, move 1 to head
              → [1:A] ⇄ [3:C] ⇄ [2:B]

put(4, D)     → evict 2 (LRU)
              → [4:D] ⇄ [1:A] ⇄ [3:C]
```

---

## 3. 구현 — Hash Map + DLL

### 자료
- Hash Map — key → node (O(1) lookup)
- Doubly Linked List — recency order
  - Head — most recent
  - Tail — least recent (eviction)

### 핵심 연산
- **Get(key)** — lookup → node → move to head → return val (O(1))
- **Put(key, val)** — 새거면 추가 + size 검사. 있으면 update + move.

### Python 구현
```python
class Node:
    def __init__(self, key, val):
        self.key = key
        self.val = val
        self.prev = None
        self.next = None

class LRUCache:
    def __init__(self, capacity):
        self.cap = capacity
        self.cache = {}        # key → node
        self.head = Node(0, 0)
        self.tail = Node(0, 0)
        self.head.next = self.tail
        self.tail.prev = self.head
    
    def _remove(self, node):
        node.prev.next = node.next
        node.next.prev = node.prev
    
    def _add_to_head(self, node):
        node.next = self.head.next
        node.prev = self.head
        self.head.next.prev = node
        self.head.next = node
    
    def get(self, key):
        if key not in self.cache:
            return -1
        node = self.cache[key]
        self._remove(node)
        self._add_to_head(node)
        return node.val
    
    def put(self, key, val):
        if key in self.cache:
            self._remove(self.cache[key])
        node = Node(key, val)
        self.cache[key] = node
        self._add_to_head(node)
        if len(self.cache) > self.cap:
            lru = self.tail.prev
            self._remove(lru)
            del self.cache[lru.key]
```

### 시간 — O(1) (get / put 둘 다)

---

## 4. Python — OrderedDict

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity):
        self.cap = capacity
        self.cache = OrderedDict()
    
    def get(self, key):
        if key not in self.cache:
            return -1
        self.cache.move_to_end(key)
        return self.cache[key]
    
    def put(self, key, val):
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = val
        if len(self.cache) > self.cap:
            self.cache.popitem(last=False)    # remove oldest
```

→ `OrderedDict` 가 hash + DLL 의 내장 구현.

---

## 5. 변형 — Eviction 정책

### LRU — Least Recently Used (기본)
- 가장 오래된 접근 evict

### LFU — Least Frequently Used
- 가장 적게 접근한 evict
- 자세히 → [[../advanced/lfu-cache]] (TBD)

### FIFO
- 먼저 들어온 순서대로 evict
- DLL 없이 — queue

### Random
- 랜덤 evict
- 간단 / 일부 사례 OK

### MRU — Most Recently Used (rare)
- 가장 최근 접근 evict
- DB scan 등 특수

### ARC (Adaptive Replacement Cache)
- LRU + LFU 결합 (IBM)
- Hit rate 좋음
- 복잡

### Clock / Second-Chance
- OS page replacement
- 비트 1 → 0, 0 → evict

### TTL (Time-to-Live)
- 시간 기반
- LRU 와 별도

---

## 6. 응용

### CPU / OS
- Page cache (Linux)
- TLB

### DB
- Buffer pool (PostgreSQL, MySQL InnoDB)

### CDN / HTTP cache
- Edge cache

### Application
- Redis (LRU 옵션)
- Memcached
- Application-level cache (Python `functools.lru_cache`)

### Python decorator
```python
from functools import lru_cache

@lru_cache(maxsize=128)
def fib(n):
    if n < 2: return n
    return fib(n-1) + fib(n-2)
```

---

## 7. Distributed LRU

### 문제
- 한 server 의 LRU — 한 cache
- 여러 server — 일관성?

### 해결
- Consistent Hashing — key → server
- 각 server 의 local LRU
- Coordination 없음

자세히 → [[../../network/topics/consistent-hashing]]

---

## 8. Cache miss / 부정확

### Cache miss penalty
- DB / origin 다시 — 비싸짐

### LRU 의 한계
- "1 회성 큰 scan" — LRU 망침
  - 모든 hot data 가 evict
  - 다음 scan 도 cache miss
- 해결 — ARC / 2Q / SLRU

### Thundering herd
- 동시 cache miss → 동시 origin fetch → origin 부담
- 해결 — singleflight (Go), request coalescing

---

## 9. Java — LinkedHashMap

```java
class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private int capacity;
    
    public LRUCache(int capacity) {
        super(capacity, 0.75f, true);    // access-order
        this.capacity = capacity;
    }
    
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }
}
```

→ `LinkedHashMap` (accessOrder=true) — 자동 LRU.

---

## 10. 함정

### 함정 1 — Thread safety
멀티 thread — race condition. Lock / concurrent.

### 함정 2 — Memory 의 GC
큰 cache — GC pause. off-heap (Ehcache / Caffeine).

### 함정 3 — Hot key
한 key 의 매우 큰 트래픽 → cache hit 단일 entry.
부담 X 지만 — distributed cache 의 hot spot.

### 함정 4 — Stale data
TTL 없으면 — 옛 데이터. Cache invalidation 필요.

### 함정 5 — 큰 value
한 entry 가 GB — eviction 불균등. Size-based limit.

### 함정 6 — Capacity 의 0 / 1
경계 — code 가 안 깨지는지 확인.

### 함정 7 — Get / Put 의 순서
Put 이 size check 후 evict — 새 entry 가 evict 대상?
일반적으로 새거 추가 → 다음 evict.

---

## 11. 학습 자료

- LeetCode 146 LRU Cache
- "Cache Replacement Policies" (Wikipedia)
- Caffeine (Java) — Window TinyLFU
- Linux MM docs (page cache)

---

## 12. 관련

- [[linked-lists]] — Hub
- [[doubly-linked-list]] — 기반
- [[../hash-tables/hash-tables]] — Hash Map
- [[../../operating-system/operating-system]] — page cache
