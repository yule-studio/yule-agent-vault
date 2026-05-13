---
title: "Cuckoo Hashing — O(1) Worst-case Lookup"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:25:00+09:00
tags:
  - data-structure
  - cuckoo
  - hash
---

# Cuckoo Hashing

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 2 hash / kick / filter |

**[[advanced|↑ Advanced]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

**2 (or k) hash 함수 + 옛 key 를 다른 자리로 밀어내기** — O(1) **worst-case** lookup. Pagh & Rodler, 2001.

---

## 2. 동작

### 2 hash function
```
hash1(key) → table1 index
hash2(key) → table2 index
```

### Lookup
```
return table1[hash1(key)] == key OR table2[hash2(key)] == key
```

→ 항상 2 위치만 검사 — O(1) 보장.

### Insert
```
candidate = key, table 1
while loop:
    if table[candidate] empty:
        place candidate
        return
    else:
        kick out old key, place candidate
        candidate = old key, other table
        loop limit — rehash 필요
```

---

## 3. 시각화

```
hash1("a") = 3, hash2("a") = 5
hash1("b") = 3, hash2("b") = 7

insert "a":
  table1[3] = "a"  ✓

insert "b":
  table1[3] = "b"  (kick a)
  → "a" 의 hash2 = 5
  table2[5] = "a"  ✓
```

---

## 4. 시간 / 공간

| | 시간 |
| --- | --- |
| **Lookup** | O(1) worst-case |
| **Insert** | O(1) amortized |
| **Delete** | O(1) |

### Load factor < 50%
- 2 table — 합쳐서 50% 까지 안전
- 더 높으면 — infinite kick 가능 → rehash

### k-ary (k tables, k=4)
- 90%+ load OK

---

## 5. Insert — 자세히

```python
def insert(key, value, table1, table2, max_kicks=500):
    for _ in range(max_kicks):
        idx1 = hash1(key) % size
        if table1[idx1] is None:
            table1[idx1] = (key, value)
            return True
        # kick
        old_key, old_value = table1[idx1]
        table1[idx1] = (key, value)
        key, value = old_key, old_value
        
        idx2 = hash2(key) % size
        if table2[idx2] is None:
            table2[idx2] = (key, value)
            return True
        old_key, old_value = table2[idx2]
        table2[idx2] = (key, value)
        key, value = old_key, old_value
    
    rehash()    # max_kicks 초과
    return insert(key, value, ...)
```

---

## 6. Rehash

### Trigger
- Max kicks 초과 → cycle 가능성

### Solution
- 새 hash 함수 / 더 큰 table
- 모든 key — 다시 insert

### 비용
- Amortized — load factor 충분 낮으면 작음

---

## 7. Cuckoo vs Chaining vs Linear Probing

| | Chaining | Linear | Cuckoo |
| --- | --- | --- | --- |
| **Lookup worst** | O(N) | O(N) | **O(1)** |
| **Insert** | O(1) | O(1) | O(1) amortized |
| **Cache** | Bad | Good | OK |
| **Load factor** | > 1 OK | < 0.7 | < 0.5 (2-way) |
| **Memory** | + 포인터 | Tight | 2-way 50% |

### Cuckoo 의 단점
- 50% load — 메모리 낭비
- Insert 가 가끔 비싸 (kick)

### 사용
- Latency-critical
- Cache (Memcached 의 옵션)

---

## 8. Cuckoo Filter

### 정의
- Bloom filter 의 대체
- Cuckoo hash 기반
- Fingerprint 저장 (key 의 짧은 hash)

### vs Bloom
| | Bloom | Cuckoo Filter |
| --- | --- | --- |
| **Delete** | X | O |
| **Memory** | 비슷 | 약간 |
| **FP rate** | 비슷 | 약간 |
| **Cache** | Bad (k random access) | Good (1-2 cache line) |
| **Lookup** | O(k) | O(1) |

자세히 → [[bloom-filter]]

---

## 9. d-ary Cuckoo

### 정의
- 2 hash → d hash
- 더 높은 load factor

### Memcached
- 1-7 hash (configurable)

---

## 10. Hopscotch Hashing

### 비슷한 아이디어
- 자기 hash 위치의 H 거리 안에 (H=32)
- Swap 으로 가까이 유지
- Linear probing 의 cache + Cuckoo 의 O(1) lookup

---

## 11. 사용

### CRC / encrypted Bloom 대체
- Cuckoo filter

### Memcached
- Cuckoo hash 옵션

### F2FS (Flash File System)
- Cuckoo hash for directory

### Network routers
- Flow table

### Disk cache
- 일부 시스템

---

## 12. 함정

### 함정 1 — Cycle / livelock
Insert 가 무한 kick. Max iter + rehash.

### 함정 2 — Load factor
50% 한계 (2-way). 80% 시도 — 자주 rehash.

### 함정 3 — Hash 의 독립성
2 hash — 독립이어야. 같은 결과 — 항상 collision.

### 함정 4 — Concurrency
Multi-thread — kick 시 lock complex. Lock-free Cuckoo 가 있음.

### 함정 5 — Resize 의 cost
2× size — 모든 key 재해시. Incremental.

### 함정 6 — Bias
일부 key — kick chain 자주 발생. 변형 (Cuckoo Bucket).

---

## 13. 학습 자료

- "Cuckoo Hashing" — Pagh, Rodler 2001
- "Cuckoo Filter" — Fan 2014
- "MemC3: Compact and Concurrent Memcache with Dumber Caching and Smarter Hashing" — Fan

---

## 14. 관련

- [[advanced]] — Hub
- [[../hash-tables/collision-resolution]] — 비교
- [[bloom-filter]] — Cuckoo Filter
