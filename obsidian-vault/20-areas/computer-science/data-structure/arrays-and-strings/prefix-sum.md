---
title: "Prefix Sum / Difference Array — 누적 합 / 차분"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T11:20:00+09:00
tags:
  - data-structure
  - prefix-sum
  - range-query
---

# Prefix Sum / Difference Array

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 1D / 2D / Diff array |

**[[arrays-and-strings|↑ Arrays & Strings]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

**누적 합 (prefix sum)** — 구간 [i, j] 의 합을 O(1) 에 조회. 한 번 전처리 O(N).
**차분 (difference array)** — 구간 [i, j] 에 값 더하기 O(1).

---

## 2. Prefix Sum — 1D

### 정의
```
arr:     [3, 1, 4, 1, 5, 9, 2, 6]
prefix:  [0, 3, 4, 8, 9, 14, 23, 25, 31]
          ↑ prefix[0] = 0
          
prefix[i] = arr[0] + arr[1] + ... + arr[i-1]
```

### 구축 — O(N)
```python
def build_prefix(arr):
    prefix = [0] * (len(arr) + 1)
    for i in range(len(arr)):
        prefix[i+1] = prefix[i] + arr[i]
    return prefix
```

### Range sum query — O(1)
```python
def range_sum(prefix, l, r):
    # arr[l..r] 의 합 (inclusive)
    return prefix[r+1] - prefix[l]
```

### 예
```python
arr = [3, 1, 4, 1, 5, 9, 2, 6]
prefix = build_prefix(arr)
range_sum(prefix, 2, 4)    # arr[2..4] = 4+1+5 = 10
                            # prefix[5] - prefix[2] = 14 - 4 = 10
```

---

## 3. 응용 — Subarray sum

### Subarray sum equals K (음수 포함)
```python
def subarray_sum(arr, k):
    prefix_count = {0: 1}    # prefix_sum → count
    s = 0
    count = 0
    for x in arr:
        s += x
        # arr[i..j] 의 합 = k
        # → prefix[j+1] - prefix[i] = k
        # → prefix[i] = prefix[j+1] - k
        if s - k in prefix_count:
            count += prefix_count[s - k]
        prefix_count[s] = prefix_count.get(s, 0) + 1
    return count
```

### 효과
- Sliding window 음수 X
- Hash 결합 — O(N)

### Equal substrings (0/1 같은 개수)
```python
def find_max_length(arr):
    # 0 → -1, 1 → 1
    prefix_index = {0: -1}
    s = 0
    result = 0
    for i, v in enumerate(arr):
        s += 1 if v == 1 else -1
        if s in prefix_index:
            result = max(result, i - prefix_index[s])
        else:
            prefix_index[s] = i
    return result
```

---

## 4. Prefix Sum — 2D

### 정의
```
matrix:
1 2 3
4 5 6
7 8 9

prefix:
0  0  0  0
0  1  3  6
0  5 12 21
0 12 27 45

prefix[i+1][j+1] = arr[i][j] + prefix[i][j+1] + prefix[i+1][j] - prefix[i][j]
```

### 구축
```python
def build_prefix_2d(matrix):
    m, n = len(matrix), len(matrix[0])
    prefix = [[0] * (n+1) for _ in range(m+1)]
    for i in range(m):
        for j in range(n):
            prefix[i+1][j+1] = matrix[i][j] + prefix[i][j+1] + prefix[i+1][j] - prefix[i][j]
    return prefix
```

### Sum of submatrix (r1, c1, r2, c2)
```python
def sum_region(prefix, r1, c1, r2, c2):
    return (prefix[r2+1][c2+1]
            - prefix[r1][c2+1]
            - prefix[r2+1][c1]
            + prefix[r1][c1])
```

### Inclusion-exclusion
```
       ┌───────┬───┐
       │       │   │
       │  C    │ D │
       ├───────┼───┤
       │  A    │ B │
       └───────┴───┘
       
       Sum(B) = (A+B+C+D) - (A+C) - (C+D) + C
                = prefix[r2+1][c2+1] - prefix[r1][c2+1] - prefix[r2+1][c1] + prefix[r1][c1]
```

---

## 5. Difference Array — 1D

### 문제
- Update — 구간 [l, r] 에 +x — naive O(N) 매번
- 여러 update 후 — 결과 봐야 함

### 차분 배열
```
arr:    [0, 0, 0, 0, 0, 0]
diff:   [0, 0, 0, 0, 0, 0, 0]    (size +1)

update(l=1, r=3, +5):
diff[1] += 5
diff[4] -= 5

→ diff: [0, 5, 0, 0, -5, 0, 0]

prefix sum of diff:
[0, 5, 5, 5, 0, 0]    ← 원하는 arr
```

### 구현
```python
def range_update(diff, l, r, x):
    diff[l] += x
    diff[r+1] -= x

def build_arr(diff):
    arr = [0] * len(diff)
    arr[0] = diff[0]
    for i in range(1, len(diff)):
        arr[i] = arr[i-1] + diff[i]
    return arr[:-1]    # last is auxiliary
```

### 효과
- 각 update — O(1)
- 최종 변환 — O(N)
- Q update + 1 query — O(Q + N) (naive: O(Q×N))

---

## 6. Difference — 2D

```python
def update_2d(diff, r1, c1, r2, c2, x):
    diff[r1][c1] += x
    diff[r1][c2+1] -= x
    diff[r2+1][c1] -= x
    diff[r2+1][c2+1] += x

def build_2d(diff):
    m, n = len(diff), len(diff[0])
    arr = [[0] * n for _ in range(m)]
    for i in range(m):
        for j in range(n):
            arr[i][j] = diff[i][j]
            if i > 0: arr[i][j] += arr[i-1][j]
            if j > 0: arr[i][j] += arr[i][j-1]
            if i > 0 and j > 0: arr[i][j] -= arr[i-1][j-1]
    return arr
```

---

## 7. 응용 — 항공편 예약 (LeetCode 1109)

```python
def corp_flight_bookings(bookings, n):
    diff = [0] * (n + 1)
    for first, last, seats in bookings:
        diff[first - 1] += seats
        diff[last] -= seats
    
    # cumulative sum
    for i in range(1, n):
        diff[i] += diff[i-1]
    return diff[:n]
```

---

## 8. Prefix Sum vs Segment Tree

| | Prefix Sum | Segment Tree |
| --- | --- | --- |
| **구축** | O(N) | O(N) |
| **Range Query** | O(1) | O(log N) |
| **Point Update** | O(N) 재구축 | O(log N) |
| **Range Update** | O(N) | O(log N) (lazy) |
| **공간** | O(N) | O(N) |
| **복잡** | 단순 | 복잡 |

### 선택
- Static (query only) — Prefix Sum
- Dynamic (update + query) — Segment Tree / Fenwick

자세히 → [[../trees/segment-tree]] (TBD)

---

## 9. Fenwick / Binary Indexed Tree

### 동적 — 점 갱신 + 구간 쿼리
- O(log N) 둘 다

자세히 → [[../advanced/fenwick-tree]] (TBD)

---

## 10. Prefix XOR / Min / Max / GCD

### Prefix 의 다른 연산
- Sum — `+ -`
- XOR — `^`
- Min / Max — 단방향만 (역연산 X)
- GCD / Product — 가능 (overflow 주의)

### XOR
```python
def range_xor(arr, l, r):
    prefix = [0]
    for x in arr:
        prefix.append(prefix[-1] ^ x)
    return prefix[r+1] ^ prefix[l]
```

→ XOR 의 self-inverse 활용.

### Min / Max — 정적만
- 구간 max — Sparse Table O(N log N) build, O(1) query

---

## 11. 함정

### 함정 1 — Index off-by-one
prefix size N+1 — index 0 부터. `arr[l..r] = prefix[r+1] - prefix[l]`.

### 함정 2 — Empty array
`prefix[0] = 0` (sentinel) — 정의 명확히.

### 함정 3 — Overflow
큰 N + 큰 값 — int 한계. long / 모듈러.

### 함정 4 — Diff 의 inclusive / exclusive
update(l, r) 의 r — `diff[r+1] -= x` (after r). r exclusive 면 다름.

### 함정 5 — Sliding window 대체 X
음수 — sliding 안 되지만 prefix + hash 가 가능.

### 함정 6 — Real number 의 정확도
부동소수 — 누적 오차. 정수 / fraction.

---

## 12. 학습 자료

- LeetCode "Prefix Sum" tag
- CLRS — segment tree / partial sums
- Competitive programming handbook (Halim)

---

## 13. 관련

- [[arrays-and-strings]] — Hub
- [[sliding-window]] — 비교
- [[../trees/segment-tree]] (TBD)
- [[../advanced/fenwick-tree]] (TBD)
