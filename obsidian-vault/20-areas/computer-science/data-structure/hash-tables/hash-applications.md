---
title: "Hash Table 응용 — 실무 사례"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T14:15:00+09:00
tags:
  - data-structure
  - hash
  - applications
---

# Hash Table 응용

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | dict / set / DB / 분산 |

**[[hash-tables|↑ Hash Tables]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 사용 — 어디 안 쓸까

Hash table 은 모던 컴퓨팅의 거의 모든 곳:
- 변수 lookup (컴파일러 symbol table)
- DB 인덱스 (Hash index)
- 캐시 (Memcached / Redis)
- 분산 시스템 (Consistent hashing)
- 보안 (Hash 비교, password)
- 자료구조 (set, multimap)

---

## 2. 알고리즘 패턴 — Hash 활용

### 2.1 Two Sum
```python
def two_sum(nums, target):
    seen = {}
    for i, x in enumerate(nums):
        if target - x in seen:
            return [seen[target - x], i]
        seen[x] = i
```

→ O(N) vs O(N²) (naive).

### 2.2 Contains Duplicate
```python
def contains_dup(nums):
    return len(set(nums)) != len(nums)
```

### 2.3 Group Anagrams
```python
def group_anagrams(strs):
    groups = defaultdict(list)
    for s in strs:
        key = "".join(sorted(s))
        groups[key].append(s)
    return list(groups.values())
```

### 2.4 Subarray sum = K (음수 포함)
```python
def subarray_sum(nums, k):
    prefix_count = {0: 1}
    s = 0
    count = 0
    for x in nums:
        s += x
        if s - k in prefix_count:
            count += prefix_count[s - k]
        prefix_count[s] = prefix_count.get(s, 0) + 1
    return count
```

→ Prefix sum + hash → O(N).

### 2.5 First non-repeating character
```python
def first_non_repeat(s):
    from collections import Counter
    count = Counter(s)
    for c in s:
        if count[c] == 1:
            return c
    return None
```

### 2.6 LRU Cache
- Hash + DLL — 자세히 [[../linked-lists/lru-cache]]

---

## 3. Set — 변형

### 정의
- Hash table — value 가 사실상 dummy
- 멤버십 검사 O(1)

### 응용
- 중복 제거
- 교집합 / 합집합 / 차집합
- 멤버십 확인

### Python
```python
s = set()
s.add(1)
1 in s
s & t   # intersect
s | t   # union
s - t   # diff
```

### Java
```java
Set<Integer> s = new HashSet<>();
Set<Integer> sorted = new TreeSet<>();    // O(log N) — BST
```

---

## 4. Bag / Multiset / Counter

### 정의
- 중복 허용 — count
- Python `collections.Counter`

```python
from collections import Counter

c = Counter([1, 2, 2, 3, 3, 3])
c[2]            # 2
c[5]            # 0 (default)
c.most_common(2)   # [(3, 3), (2, 2)]
```

### 사용
- 단어 빈도
- Anagram (글자 분포 비교)

---

## 5. DefaultDict / DefaultMap

```python
from collections import defaultdict

d = defaultdict(list)
d['a'].append(1)    # 자동 — d['a'] = []
d['a'].append(2)
# d == {'a': [1, 2]}
```

### Java
```java
map.computeIfAbsent(k, x -> new ArrayList<>()).add(v);
```

### Go
```go
m[k] = append(m[k], v)    // nil slice 도 append 가능
```

---

## 6. DB — Hash Index

### Hash index vs B-tree index
| | Hash | B-tree |
| --- | --- | --- |
| **=** 검색 | O(1) | O(log N) |
| **<, >, BETWEEN** | X | O(log N) |
| **정렬** | X | O(N) |
| **공간** | 작음 | 큼 |

### 사용
- PostgreSQL — Hash Index (사용 잘 안 됨)
- Redis — Hash 자료구조
- DynamoDB — Hash partition key (Equality only)

자세히 → database/index

---

## 7. Memcached — In-memory KV

### 핵심
- 분산 hash table
- LRU eviction
- Consistent hashing 으로 분산

### 사용
- DB 캐시
- Session store
- 카운터

---

## 8. Redis — Hash 자료구조

### Type
- `HSET`, `HGET`, `HGETALL`
- 한 key 안에 — 여러 field-value

```
HSET user:1 name alice email alice@a.com age 30
HGET user:1 name
HGETALL user:1
```

### 내부 — ziplist (작음) or hashtable
- 작은 hash — ziplist (linear scan)
- 큰 — hashtable

자세히 → database/redis/data-types

---

## 9. Consistent Hashing

### 정의
- 분산 시스템의 hash
- 노드 변경 시 — 영향 1/N

### 사용
- Memcached client (Ketama)
- DynamoDB / Cassandra
- LB

자세히 → [[../../network/topics/consistent-hashing]]

---

## 10. Bloom Filter — Probabilistic

### 정의
- Hash table 의 변형
- Membership — "있을 수도 있음" vs "절대 없음"
- 매우 작은 메모리

### 사용
- Cache 우회 (없으면 DB 조회 X)
- Cassandra / LevelDB / RocksDB

자세히 → [[../advanced/bloom-filter]] (TBD)

---

## 11. Cuckoo Hashing / Cuckoo Filter

### 정의
- Open addressing 의 변형
- O(1) worst-case lookup
- 변형 Filter — Bloom 대체

자세히 → [[../advanced/cuckoo-hashing]] (TBD)

---

## 12. 컴파일러 / 인터프리터

### Symbol Table
- 변수 / 함수 이름 → 정보
- Scope 별 nested
- 모든 컴파일러의 핵심

### Interning (String pool)
- 같은 string 한 객체

### Method dispatch
- Class 의 method table
- Python — dict
- Java — vtable + hash 일부

---

## 13. JSON / XML parsing

### 객체 → Hash table
```javascript
const obj = JSON.parse('{"name": "alice"}');
obj.name;     // O(1) lookup
```

### Insertion order
- 모던 — 보장 (Python 3.6+ / ES2015+)

---

## 14. 함정

### 함정 1 — Mutable key
List / dict — hash 불가 또는 (mutable hash 시) lookup 깨짐.
Tuple / frozenset / 사용자 immutable.

### 함정 2 — `dict.keys()` 의 iteration 중 modification
RuntimeError. List(d.keys()) 로 복사 후.

### 함정 3 — Worst-case complexity
이론적 O(1) 평균. Worst case O(N). 적의 입력 / DoS.

### 함정 4 — 메모리 fragmentation
큰 dict — 메모리 흩어짐. 데이터 packing.

### 함정 5 — Hash collision attack
Random seed 없음 → DoS. 모던 — 자동.

### 함정 6 — Sorted iteration 필요 시
Hash — 순서 없음. TreeMap / SortedDict.

### 함정 7 — 큰 hash table 의 자원
1B entries — 수 GB. 분산 / 외부 저장.

---

## 15. 학습 자료

- CLRS Ch 11
- "Designing Data-Intensive Applications" — Kleppmann
- Redis docs (hash 자료구조)

---

## 16. 관련

- [[hash-tables]] — Hub
- [[hash-functions]] / [[collision-resolution]] / [[resizing-rehashing]]
- [[../linked-lists/lru-cache]] — Hash + DLL
- [[../../network/topics/consistent-hashing]]
