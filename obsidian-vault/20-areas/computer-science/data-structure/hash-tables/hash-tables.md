---
title: "해시 테이블 (Hash Tables)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T07:00:00+09:00
tags:
  - data-structure
  - hash-table
  - hash-map
---

# 해시 테이블 (Hash Tables)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 수준 + 11 언어 + 깊이 |

**[[../data-structure|↑ data-structure]]** · **[[../../computer-science|↑↑ computer-science]]**

---

## 1. 한 줄 정의

**키를 해시 함수로 정수 인덱스로 변환**해 배열에 저장. **평균 O(1)** 의
삽입·검색·삭제. dict / Map / HashMap / `unordered_map` 의 내부.

---

## 2. 개념의 깊이

### 2.1 왜 O(1) 인가 — 직접 주소화의 일반화

배열은 인덱스로 O(1). 문제: 키가 정수가 아닐 때 (문자열, 객체, 튜플).

**해시 함수** `h(key) → integer in [0, M-1]` 가 임의 키를 배열 인덱스로 매핑:

```
"apple"  →  hash("apple")  →  37  →  table[37]
"banana" →  hash("banana") →  12  →  table[12]
"cherry" →  hash("cherry") →  37  →  ⚠️ 충돌!
```

이론적으로 키가 모두 다른 인덱스에 매핑되면 O(1). 현실은 **충돌 (collision)**.

### 2.2 충돌 해결 — 두 학파

#### Chaining (분리 연결)
각 버킷에 **연결 리스트** (또는 배열, 트리). 충돌 시 그 리스트에 추가.

```
table[37] → ["apple", "cherry"]
table[12] → ["banana"]
```

- **장점**: 단순 / 삭제 쉬움 / load factor > 1 가능.
- **단점**: 캐시 비친화 (포인터 따라감) / 메모리 오버헤드.
- **Java HashMap** (8개 이상 충돌 시 트리로 전환), **Python dict 의 이전 구현**.

#### Open Addressing (개방 주소법)
충돌 시 다른 빈 슬롯을 **probing** 으로 찾음.

```
충돌 시:
  Linear: h(k), h(k)+1, h(k)+2, ...
  Quadratic: h(k), h(k)+1², h(k)+2², ...
  Double hashing: h(k), h(k) + h2(k), h(k) + 2*h2(k), ...
```

- **장점**: 캐시 친화 (인접 슬롯) / 메모리 효율 / 포인터 없음.
- **단점**: 삭제 복잡 (tombstone), load factor < 0.7 필요.
- **Python dict 의 현재 구현 (PEP 412)**, **Rust HashMap (Swiss Table)**.

### 2.3 좋은 해시 함수의 조건

1. **균등 분포 (Uniform)** — 키가 인덱스에 고르게 분포.
2. **빠름** — 짧은 시간에 계산 가능.
3. **결정적 (Deterministic)** — 같은 키 → 항상 같은 해시.
4. **눈사태 효과 (Avalanche)** — 키 한 비트 바뀌면 해시 절반 비트 바뀜.

수학적 깊이:

- **Modular hashing**: `h(k) = k mod M`. 빠르지만 M 이 2의 거듭제곱이면 하위 비트만 사용 (편향).
- **Multiplicative**: `h(k) = ⌊M × (kA mod 1)⌋` (Knuth: A = (√5 - 1) / 2 ≈ 0.618 황금비).
- **Universal Hashing (Carter & Wegman, 1979)** — 함수 집합에서 랜덤 선택 → 적대적 공격 방어.
- **MurmurHash, xxHash, CityHash** — 현대 비암호화 해시. 매우 빠름.
- **SipHash (DJB)** — 해시 충돌 공격 (HashDoS) 방어용. Python 3.4+ / Rust 기본.

### 2.4 Load Factor (적재율)

```
load_factor = N (원소 수) / M (버킷 수)
```

- Chaining: 0.75 ~ 1.0 (Java HashMap 기본 0.75 → 넘으면 resize)
- Open Addressing: 0.5 ~ 0.7 (Python 0.66, Rust 0.875 with SIMD)

**Resize**: load factor 초과 시 M 을 2 배 + 모든 원소 재해시 → amortized O(1).

### 2.5 해시 충돌 공격 (Hash DoS)

악의적 사용자가 같은 해시값을 갖는 키를 N 개 입력 → 모든 키가 한 버킷 → O(N²).
2003 년 한국에서 발견된 PHP / Java / Python 해시 DoS 가 대표 사례.

방어:
- 프로세스 시작 시 **랜덤 seed** (Python 3.3+, Java 1.7+).
- 적대적 환경에서 **SipHash** 사용.
- 트리 fallback (Java HashMap 8 개 이상 → red-black tree).

### 2.6 역사

- **1953 IBM** — Hans Peter Luhn 의 인터널 메모. 첫 해시 테이블 아이디어.
- **1968 Knuth TAOCP Vol. 3** — 해시 테이블의 표준 분석.
- **1970s** — 다양한 충돌 해결 (chaining vs probing) 연구.
- **2017 Swiss Table** (Google) — SIMD + 메타데이터 분리. C++ Abseil / Rust HashMap.
- **2018 robin_hood** — Robin Hood Hashing. 균등 probing 거리.

---

## 3. 핵심 직관 — Chaining 그림

```
buckets (M=8):

idx 0: → []
idx 1: → [("dog", 5)] → []
idx 2: → [("cat", 3)] → [("act", 7)] → []   ← 충돌
idx 3: → []
idx 4: → [("bird", 9)] → []
idx 5: → []
idx 6: → [("fish", 1)] → []
idx 7: → []

h("cat") = 2, h("act") = 2 → 같은 버킷 chain
```

조회: `h(key)` 로 버킷 찾고, 그 안 리스트에서 키 비교 (보통 1-2 개).

---

## 4. Open Addressing 그림

```
buckets (M=8):

idx 0: ("dog", 5)
idx 1: ("cat", 3)
idx 2: ("act", 7)   ← h("act") = 1, but idx 1 occupied → linear probe to idx 2
idx 3: empty
idx 4: ("bird", 9)
idx 5: empty
idx 6: ("fish", 1)
idx 7: empty
```

삭제 시: 즉시 비우면 probing 체인 끊김. **Tombstone** (삭제 마커) 표시.

---

## 5. 복잡도

| 연산 | 평균 | 최악 (해시 충돌) | 비고 |
| --- | --- | --- | --- |
| Insert | **O(1)** | O(N) | resize 시 amortized O(N) but 1번 |
| Search | **O(1)** | O(N) | |
| Delete | **O(1)** | O(N) | Open addressing 은 tombstone |
| 순회 | O(N) | O(M+N) | empty 버킷도 봐야 |
| 공간 | O(N) | O(M) ≥ O(N) | load factor 로 결정 |

**현실**: 좋은 해시 + 적절한 load factor 면 거의 항상 O(1).
**Java HashMap (Java 8+)**: 한 버킷 8개 초과 시 트리로 → 최악 O(log N).

---

## 6. 의사 코드

### Chaining
```
TABLE[0..M-1] 각각 empty list

INSERT(key, value):
    i ← h(key)
    if key exists in TABLE[i]: update value
    else: TABLE[i].append((key, value))
    if size / M > threshold: resize()

SEARCH(key):
    for (k, v) in TABLE[h(key)]:
        if k == key: return v
    return None

DELETE(key):
    remove (key, _) from TABLE[h(key)]
```

### Open Addressing (Linear Probing)
```
INSERT(key, value):
    i ← h(key)
    while TABLE[i] is occupied and TABLE[i].key != key:
        i ← (i + 1) % M
    TABLE[i] ← (key, value)

SEARCH(key):
    i ← h(key)
    while TABLE[i] is occupied:
        if TABLE[i].key == key: return TABLE[i].value
        i ← (i + 1) % M
    return None
```

---

## 7. 언어별 구현 (11 언어)

### Python 3
```python
# dict (Open Addressing + 랜덤 seed)
d = {}
d["apple"] = 5
v = d.get("apple", -1)
del d["apple"]
"apple" in d

# defaultdict — 기본값 자동
from collections import defaultdict
counts = defaultdict(int)
for c in "banana": counts[c] += 1

# Counter — 빈도
from collections import Counter
c = Counter("hello")   # {'l': 2, 'h': 1, 'e': 1, 'o': 1}
c.most_common(2)

# set
s = {1, 2, 3}
s.add(4); s.discard(1)
2 in s    # O(1)

# 직접 구현 (학습용 — Chaining)
class HashMap:
    def __init__(self, capacity=16):
        self.capacity = capacity
        self.size = 0
        self.buckets = [[] for _ in range(capacity)]
    
    def _hash(self, key):
        return hash(key) % self.capacity
    
    def put(self, key, value):
        idx = self._hash(key)
        for i, (k, v) in enumerate(self.buckets[idx]):
            if k == key:
                self.buckets[idx][i] = (key, value)
                return
        self.buckets[idx].append((key, value))
        self.size += 1
        if self.size > self.capacity * 0.75:
            self._resize()
    
    def get(self, key, default=None):
        for k, v in self.buckets[self._hash(key)]:
            if k == key: return v
        return default
    
    def _resize(self):
        old = self.buckets
        self.capacity *= 2
        self.buckets = [[] for _ in range(self.capacity)]
        self.size = 0
        for bucket in old:
            for k, v in bucket: self.put(k, v)
```

### Java
```java
import java.util.*;
HashMap<String, Integer> map = new HashMap<>();
map.put("apple", 5);
map.getOrDefault("apple", -1);
map.remove("apple");

// LinkedHashMap — 삽입 순서 유지
LinkedHashMap<String, Integer> ordered = new LinkedHashMap<>();

// TreeMap — 정렬 (Red-Black tree, O(log N))
TreeMap<String, Integer> sorted = new TreeMap<>();

HashSet<Integer> set = new HashSet<>();
set.add(1); set.contains(1);

// Concurrent
ConcurrentHashMap<String, Integer> concurrent = new ConcurrentHashMap<>();
```

### C++
```cpp
#include <unordered_map>
#include <unordered_set>
#include <map>

std::unordered_map<std::string, int> m;
m["apple"] = 5;
auto it = m.find("apple");
m.erase("apple");

std::unordered_set<int> s = {1, 2, 3};

// 정렬된 (Red-Black tree, O(log N))
std::map<std::string, int> tm;
```

### C — 직접 구현 (libc 에 없음)
```c
#define M 1024
typedef struct Entry { char *key; int value; struct Entry *next; } Entry;
Entry *table[M] = {NULL};

unsigned hash(const char *s) {
    unsigned h = 0;
    for (; *s; s++) h = h * 31 + *s;
    return h % M;
}

void put(const char *key, int value) {
    unsigned i = hash(key);
    for (Entry *e = table[i]; e; e = e->next) {
        if (strcmp(e->key, key) == 0) { e->value = value; return; }
    }
    Entry *e = malloc(sizeof(Entry));
    e->key = strdup(key); e->value = value; e->next = table[i];
    table[i] = e;
}
```

### JavaScript
```javascript
// Map — 임의 키 가능
const m = new Map();
m.set("apple", 5);
m.get("apple");
m.delete("apple");

// Object — 문자열 / Symbol 키만
const obj = { apple: 5 };

// Set
const s = new Set([1, 2, 3]);
s.has(2);

// 주의: Map 은 삽입 순서 유지. Object 는 spec 상 어느 정도 유지.
```

### TypeScript
```typescript
const m = new Map<string, number>();
m.set("apple", 5);
const v: number | undefined = m.get("apple");

const s = new Set<number>([1, 2, 3]);

// Record<K, V> 타입
type Counts = Record<string, number>;
```

### Kotlin
```kotlin
val m = mutableMapOf<String, Int>()
m["apple"] = 5
m["apple"]                  // 5
m.remove("apple")

val s = mutableSetOf(1, 2, 3)
s.add(4)

// 정렬된 — TreeMap (Java)
val sorted = java.util.TreeMap<String, Int>()

// Immutable
val immutable = mapOf("a" to 1, "b" to 2)
```

### Swift
```swift
var m: [String: Int] = [:]
m["apple"] = 5
let v = m["apple"]          // Optional(5)
m.removeValue(forKey: "apple")

var s: Set<Int> = [1, 2, 3]
s.insert(4)
```

### Go
```go
m := make(map[string]int)
m["apple"] = 5
v, ok := m["apple"]         // 두 값 반환 (존재 확인)
delete(m, "apple")
for k, v := range m { ... }

// Set 은 map[string]struct{}{}
s := map[string]struct{}{}
s["apple"] = struct{}{}
_, ok := s["apple"]
```

### Rust
```rust
use std::collections::{HashMap, HashSet, BTreeMap};
let mut m: HashMap<String, i32> = HashMap::new();
m.insert("apple".to_string(), 5);
let v = m.get("apple");
m.remove("apple");

let mut s: HashSet<i32> = HashSet::new();
s.insert(1);

// 정렬된 (B-Tree, O(log N))
let mut tm: BTreeMap<String, i32> = BTreeMap::new();

// entry API — 패턴
*m.entry("count".to_string()).or_insert(0) += 1;
```

### C#
```csharp
using System.Collections.Generic;

var m = new Dictionary<string, int>();
m["apple"] = 5;
int v;
if (m.TryGetValue("apple", out v)) { }

var s = new HashSet<int> { 1, 2, 3 };
s.Add(4);

// SortedDictionary (Red-Black tree)
var sorted = new SortedDictionary<string, int>();
```

---

## 8. 변형 / 응용

### (a) Counter / Frequency Map
```python
from collections import Counter
freq = Counter("banana")    # b:1, a:3, n:2
freq.most_common(1)         # [('a', 3)]
```

### (b) Set 연산
교집합 / 합집합 / 차집합:
```python
a & b    # 교집합
a | b    # 합집합
a - b    # 차집합
a ^ b    # 대칭 차집합
```

### (c) LRU Cache — HashMap + Doubly Linked List
Hash 로 O(1) 찾기 + List 로 O(1) 순서 갱신.

### (d) Two Sum — 해시로 O(N)
```python
def two_sum(nums, target):
    seen = {}
    for i, x in enumerate(nums):
        if target - x in seen: return [seen[target-x], i]
        seen[x] = i
```

### (e) Bloom Filter — 확률적 set
여러 해시 함수 + 비트 배열. False positive 가능, false negative 없음. 대용량 키 존재 여부 빠르게.

### (f) Count-Min Sketch — 확률적 빈도
스트림에서 메모리 적게 빈도 추정. 네트워크 / DB 통계.

### (g) Consistent Hashing
분산 캐시 / Redis Cluster — 노드 추가/삭제 시 키 재할당 최소화.

### (h) Cuckoo Hashing
두 해시 함수. 충돌 시 기존 키를 다른 자리로 "쫓아냄" → worst-case O(1) lookup.

### (i) Perfect Hashing
컴파일 타임 키 알 때 → 충돌 0. 시간/공간 모두 O(N).

---

## 9. 함정 / 안티패턴

### 함정 1 — mutable key
```python
# 잘못 — list 는 hashable X
d = {[1, 2]: "value"}   # TypeError

# 옳음 — tuple 은 hashable
d = {(1, 2): "value"}
```

### 함정 2 — 객체의 `__hash__` / `equals` 불일치
```python
class Point:
    def __init__(self, x, y): self.x, self.y = x, y
    def __eq__(self, other): return self.x == other.x and self.y == other.y
    # __hash__ 정의 안 하면 dict 키로 사용 불가
    def __hash__(self): return hash((self.x, self.y))
```

Java/C++ 도 같음: `equals` 와 `hashCode` 일관성. 안 맞으면 set/map 에서 오작동.

### 함정 3 — Java 의 `int[]` 를 HashMap 키로
배열은 `hashCode` 가 객체 identity. `Arrays.hashCode(arr)` 직접 사용.

### 함정 4 — Python 3.7 이전 dict 순서
Python 3.6+ 부터 삽입 순서 보존 (CPython 구현 detail), 3.7 부터 spec 보장.
이전 버전 호환 코드에는 `OrderedDict` 사용.

### 함정 5 — Resize 동안 동시 수정
**concurrent modification exception** — 순회 중 수정 금지. `ConcurrentHashMap` 또는 사본 사용.

### 함정 6 — 해시 충돌 공격 (HashDoS)
공격자 입력에 신뢰할 수 없는 데이터 → SipHash 또는 트리 fallback.

### 함정 7 — Open Addressing 의 tombstone
삭제를 일반 빈 슬롯으로 표시하면 probing chain 끊김. 별도 마커 필요.

### 함정 8 — Go map 의 비결정적 순회
순회 순서가 매번 다름. **순서 의존 시 정렬 필요**.

### 함정 9 — 해시 함수가 너무 느림
SHA-256 같은 암호 해시는 hashmap 에 부적합. MurmurHash / xxHash 같은 빠른 해시 사용.

### 함정 10 — 너무 작은 초기 capacity
N=10^5 원소 예상인데 capacity=16 으로 시작 → 13 번 resize. 미리 `HashMap<>(expectedSize * 4/3)`.

---

## 10. 실전 절차

1. **신호 인식** — "빠른 조회 / 빈도 / 중복 / 매핑" → 해시 테이블.
2. **set vs map** — 키만 / 키-값?
3. **순서 필요?** — `LinkedHashMap` (Java), `OrderedDict` (Python).
4. **정렬 필요?** — `TreeMap` / `BTreeMap` / `std::map` (O(log N)).
5. **키가 hashable?** — Python: immutable 만. Java: `equals/hashCode`.
6. **초기 capacity** — 예상 크기 × 1.5 정도로.

---

## 11. 대표 문제

### Q1. Two Sum (LeetCode 1)
위 코드 참조.

### Q2. Group Anagrams (LeetCode 49)
```python
from collections import defaultdict
def group_anagrams(strs):
    groups = defaultdict(list)
    for s in strs:
        key = ''.join(sorted(s))
        groups[key].append(s)
    return list(groups.values())
```

### Q3. Top K Frequent Elements (LeetCode 347)
```python
from collections import Counter
import heapq
def top_k(nums, k):
    counts = Counter(nums)
    return [x for x, _ in counts.most_common(k)]
```

### Q4. Longest Substring Without Repeating (LeetCode 3)
```python
def length_of_longest(s):
    seen = {}
    start = best = 0
    for i, c in enumerate(s):
        if c in seen and seen[c] >= start:
            start = seen[c] + 1
        seen[c] = i
        best = max(best, i - start + 1)
    return best
```

### Q5. Contains Duplicate (LeetCode 217)
```python
def contains_dup(nums):
    return len(set(nums)) != len(nums)
```

### Q6. LRU Cache (LeetCode 146)
- 위 변형 (c) 참조.

### Q7. Word Pattern (LeetCode 290) — 양방향 매핑
```python
def word_pattern(pattern, s):
    words = s.split()
    if len(pattern) != len(words): return False
    p2w, w2p = {}, {}
    for p, w in zip(pattern, words):
        if p in p2w and p2w[p] != w: return False
        if w in w2p and w2p[w] != p: return False
        p2w[p] = w; w2p[w] = p
    return True
```

---

## 12. 연습 문제

### 입문
- [LeetCode 1 Two Sum](https://leetcode.com/problems/two-sum/)
- [LeetCode 217 Contains Duplicate](https://leetcode.com/problems/contains-duplicate/)
- [LeetCode 242 Valid Anagram](https://leetcode.com/problems/valid-anagram/)

### 중급
- [LeetCode 49 Group Anagrams](https://leetcode.com/problems/group-anagrams/)
- [LeetCode 347 Top K Frequent](https://leetcode.com/problems/top-k-frequent-elements/)
- [LeetCode 3 Longest Substring Without Repeat](https://leetcode.com/problems/longest-substring-without-repeating-characters/)
- [백준 7785 회사에 있는 사람](https://www.acmicpc.net/problem/7785)

### 고급
- [LeetCode 146 LRU Cache](https://leetcode.com/problems/lru-cache/)
- [LeetCode 460 LFU Cache](https://leetcode.com/problems/lfu-cache/)
- [LeetCode 432 All O`one Data Structure](https://leetcode.com/problems/all-oone-data-structure/)

---

## 13. 학습 자료

- **CLRS Ch.11** — 해시 테이블 표준 학술
- **Knuth TAOCP Vol. 3** — 깊은 분석
- **Universal Hashing 논문** (Carter & Wegman, 1979)
- **Swiss Table 발표** (Google, CppCon 2017) — [YouTube](https://www.youtube.com/watch?v=ncHmEUmJZf4)
- **Python dict 구현 (PEP 412)** — [docs](https://peps.python.org/pep-0412/)
- **Robin Hood Hashing** — [블로그](https://codecapsule.com/2013/11/11/robin-hood-hashing/)
- [VisuAlgo — Hash Table](https://visualgo.net/en/hashtable)

---

## 14. 관련

- [[../arrays-and-strings/arrays-and-strings]] — 해시는 배열 + 함수
- [[../linked-lists/linked-lists]] — Chaining 의 내부
- [[../../algorithm/sorting/sorting]] — bucket sort 는 해시 변형
- [[../data-structure|↑ data-structure]]
