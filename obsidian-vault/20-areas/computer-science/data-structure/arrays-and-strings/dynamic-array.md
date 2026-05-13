---
title: "Dynamic Array — 가변 크기 배열"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T11:00:00+09:00
tags:
  - data-structure
  - array
  - amortized
---

# Dynamic Array — 가변 크기 배열

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | growth / amortized / 언어별 구현 |

**[[arrays-and-strings|↑ Arrays & Strings]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

**고정 크기 배열을 자동으로 확장** 하는 자료구조. Append O(1) amortized.
Python `list`, Java `ArrayList`, C++ `vector`, Go `slice`, Rust `Vec`, JS `Array`.

---

## 2. 핵심 아이디어

```
capacity 4, size 2:    [A][B][_][_]
                              ↑ size

append C:              [A][B][C][_]
append D:              [A][B][C][D]    (가득)
append E:              capacity 부족 → 2배로 (8)
                        → 새 메모리 할당 → [A][B][C][D] 복사
                        → 새 메모리에 [E] 추가
                        [A][B][C][D][E][_][_][_]
```

### 3 가지 필드
- **data** — 실제 메모리 포인터
- **size** — 사용 중인 원소 수
- **capacity** — 할당된 메모리 크기

### 핵심
- `size <= capacity`
- size == capacity 시 — **resize** (보통 2배)
- 새 메모리 할당 → 복사 → 옛 메모리 해제

---

## 3. 시간 복잡도

| 연산 | 평균 | 최악 | 비고 |
| --- | --- | --- | --- |
| Access `a[i]` | O(1) | O(1) | 인덱스 계산 |
| Search | O(N) | O(N) | 선형 검색 |
| Append (push back) | **O(1) amortized** | O(N) | resize 시 O(N) |
| Insert at i | O(N) | O(N) | 뒤 원소 shift |
| Delete at i | O(N) | O(N) | 뒤 원소 shift |
| Pop back | O(1) | O(1) | size-- |

---

## 4. Amortized O(1) — 왜?

### 증명
- N append → 총 비용?

```
size 1 → 2 (1 복사)
size 2 → 4 (2 복사)
size 4 → 8 (4 복사)
...
size 2^(k-1) → 2^k (2^(k-1) 복사)

총 복사: 1 + 2 + 4 + ... + 2^(k-1) = 2^k - 1 ≈ N
```

- N append → N 번의 직접 + ≈ N 번 복사 = **2N 작업**
- Per append — 평균 2 = **O(1) amortized**

### 만약 1.5 배 / 2 배 / 다른 factor?
- 2 배 — 메모리 낭비 가능 (max 50% empty)
- 1.5 배 — 옛 메모리 재사용 가능 (Facebook FBVector)
- 황금비 — 이론상 best

---

## 5. 언어별 구현

### Python `list`
```python
# CPython source: Objects/listobject.c
# 처음 4, 증가 패턴: 0, 4, 8, 16, 24, 32, 40, 52, 64, ...
# 약 1.125 배 (작게)

a = []
a.append(1)        # O(1) amortized
a.pop()            # O(1)
a.insert(0, x)     # O(N) — 모든 원소 shift
a.extend([1,2,3])  # O(k)
len(a)             # O(1)
```

#### 메모리
- PyObject 포인터의 배열 (각 원소가 객체 reference)
- 같은 int 도 — int box 8 byte + pointer 8 byte = 16 byte 이상

### Java `ArrayList`
```java
ArrayList<Integer> list = new ArrayList<>(10);    // initial capacity
list.add(1);                                       // O(1) amortized

// 내부 — Object[] elementData
// growth: capacity + (capacity >> 1) = 1.5x
```

### C++ `vector`
```cpp
std::vector<int> v;
v.reserve(100);    // capacity 100 (size 0)
v.push_back(1);    // O(1) amortized
v.size();          // 사용 중
v.capacity();      // 할당

// 보통 2x growth (구현 마다)
// GCC: 2x, MSVC: 1.5x, Clang: 2x
```

#### `reserve` vs `resize`
- `reserve(n)` — capacity n (size 안 변경)
- `resize(n)` — size n (필요시 default 채움)

### Go `slice`
```go
s := make([]int, 0, 10)    // len 0, cap 10
s = append(s, 1)            // O(1) amortized

// Growth: < 1024 — 2x, > 1024 — 1.25x (대략)
// runtime/slice.go: growslice
```

#### Slice header
```go
type slice struct {
    array unsafe.Pointer
    len   int
    cap   int
}
```

### Rust `Vec<T>`
```rust
let mut v: Vec<i32> = Vec::with_capacity(10);
v.push(1);                   // O(1) amortized

// 보통 2x growth
// 정확히: max(old * 2, old + new_len)
```

### JavaScript `Array`
```javascript
const a = [];
a.push(1);                   // O(1) amortized
a[1000] = 1;                 // 희소 배열 가능

// V8: PACKED_SMI / PACKED / HOLEY
// 종류 따라 다른 표현 (smi-only / object / sparse)
```

#### Hidden class / map (V8)
- 같은 형식만 들어가면 — 효율적
- mixed type — slower

### Kotlin / Swift / C#
- 비슷 — 내부 dynamic array

---

## 6. Capacity 정책

### Two-power doubling
- 가장 흔함 (vector / ArrayList)
- 메모리 낭비 max 50%
- 옛 영역 — 새 영역에 들어갈 만큼 작음 → reuse 어려움

### 1.5x (golden-ratio-ish)
- Facebook FBVector
- 옛 영역 — 합쳐서 재할당 가능

### Fixed growth
- 매번 +N — O(N²) total (avoid)

### Custom (Python)
- 작은 list — 작게 (4, 8, 16, ...)
- 큰 list — 1.125배 정도

---

## 7. Shrink — 줄이기

### 자동 shrink 안 함
- 대부분 — pop / delete 시 capacity 안 줄임
- 메모리 다시 차지

### 명시
- C++ `shrink_to_fit()` — 가능 (request, 강제 X)
- Go — `s = s[:n]` — 같은 backing array
- Python — list 자동 shrink (큰 비율 비면)

---

## 8. Cache locality

### 장점
- 연속 메모리 → cache hit
- Prefetching 친화
- SIMD 가능

### 비교 — Linked List
- Linked list — 포인터 따라가기 → cache miss
- Array — 매우 빠름 (수~수십 배)

### 측정 (Bjarne Stroustrup 강의)
- N 원소 linked list 검색 vs vector 검색
- 모든 N (수십만까지) — vector 빠름

---

## 9. 함정

### 함정 1 — Insert at 0
모든 원소 shift — O(N). Deque / Linked List 권장.

### 함정 2 — 큰 리스트의 메모리
1B 원소 × 8 byte pointer = 8 GB. 다른 자료구조 / 외부 저장.

### 함정 3 — Reference / value semantics
Python list 의 element — reference. `[[]] * 3` — 같은 inner list 3 번.

```python
a = [[]] * 3      # 같은 list 3 번 (BAD)
a[0].append(1)
# a == [[1], [1], [1]]

a = [[] for _ in range(3)]   # 다른 list 3 번 (GOOD)
```

### 함정 4 — Iteration 중 변경
대부분 언어 — 정의되지 않은 동작. Copy 후 변경 또는 별도 collect.

### 함정 5 — Capacity 의 동시성
멀티 thread — resize 중 race condition. Concurrent collection 권장.

### 함정 6 — 자동 boxing (Java)
`ArrayList<Integer>` — Integer 객체 (boxing). `int[]` 가 빠름.

### 함정 7 — Iterator invalidation (C++)
push_back → resize → iterator dangling. Reserve 미리.

---

## 10. 실무 — 최적화

### Reserve 미리
```cpp
std::vector<int> v;
v.reserve(N);            // N 만큼 할당 — resize X
for (int i = 0; i < N; i++) v.push_back(i);
```

### 작은 inline (small buffer optimization)
- Boost `small_vector` — 작은 size 는 inline (heap 없음)
- LLVM ADT `SmallVector`

### Bucket allocator
- 큰 vector 의 할당 — pool / arena 활용
- jemalloc / tcmalloc

---

## 11. 학습 자료

- CLRS 17.4 — Amortized analysis
- "Effective C++" (Meyers) — vector 활용
- CPython source — listobject.c
- Bjarne Stroustrup — "Why you should avoid Linked Lists" 강의

---

## 12. 관련

- [[arrays-and-strings]] — Hub
- [[../linked-lists/linked-lists]] — 비교
- [[../../hardware/memory/dram-cell]] — cache / locality
