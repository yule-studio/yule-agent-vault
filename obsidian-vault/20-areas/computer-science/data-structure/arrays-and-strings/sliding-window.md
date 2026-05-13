---
title: "Sliding Window — 슬라이딩 윈도우"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T11:15:00+09:00
tags:
  - data-structure
  - pattern
  - sliding-window
---

# Sliding Window — 슬라이딩 윈도우

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Fixed / Variable / 응용 |

**[[arrays-and-strings|↑ Arrays & Strings]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

연속된 부분 (subarray / substring) 의 **window 를 움직이며** 조건 만족 / 최적화. O(N).

---

## 2. 2 가지 변형

### 2.1 Fixed-size window
```
window size k:
[a, b, c, d, e, f]
[─────]              first
   [─────]           shift 1
      [─────]        shift 2
```

### 2.2 Variable-size window
```
조건 만족 동안 확장 / 못 만족 시 축소:
[─]                  start
[───]                expand
[─────]              expand (조건 X)
   [───]             contract
```

---

## 3. Fixed-size 예 — Max sum of k consecutive

### Naive O(N×k)
```python
def max_sum_k(arr, k):
    result = float('-inf')
    for i in range(len(arr) - k + 1):
        result = max(result, sum(arr[i:i+k]))
    return result
```

### Sliding window O(N)
```python
def max_sum_k(arr, k):
    window_sum = sum(arr[:k])
    max_sum = window_sum
    for i in range(k, len(arr)):
        window_sum += arr[i] - arr[i-k]    # add new, remove old
        max_sum = max(max_sum, window_sum)
    return max_sum
```

---

## 4. Variable-size 템플릿

```python
def sliding_window(arr, condition):
    left = 0
    result = ...
    state = ...
    
    for right in range(len(arr)):
        # 1. expand: add arr[right] to state
        state += arr[right]
        
        # 2. shrink while condition violated
        while not_valid(state):
            state -= arr[left]
            left += 1
        
        # 3. update result
        result = max(result, right - left + 1)
    
    return result
```

---

## 5. 흔한 문제

### 5.1 Longest substring without repeating
```python
def length_of_longest_substring(s):
    seen = {}
    left = 0
    max_len = 0
    for right, c in enumerate(s):
        if c in seen and seen[c] >= left:
            left = seen[c] + 1
        seen[c] = right
        max_len = max(max_len, right - left + 1)
    return max_len
```

### 5.2 Minimum window substring
```python
def min_window(s, t):
    if not t: return ""
    need = Counter(t)
    have = Counter()
    have_count = 0
    need_count = len(need)
    
    left = 0
    result = [-1, -1]
    min_len = float('inf')
    
    for right, c in enumerate(s):
        have[c] += 1
        if c in need and have[c] == need[c]:
            have_count += 1
        
        while have_count == need_count:
            if right - left + 1 < min_len:
                min_len = right - left + 1
                result = [left, right]
            
            have[s[left]] -= 1
            if s[left] in need and have[s[left]] < need[s[left]]:
                have_count -= 1
            left += 1
    
    return s[result[0]:result[1]+1] if min_len != float('inf') else ""
```

### 5.3 Longest repeating character replacement
```python
def character_replacement(s, k):
    count = Counter()
    left = 0
    max_freq = 0
    result = 0
    
    for right in range(len(s)):
        count[s[right]] += 1
        max_freq = max(max_freq, count[s[right]])
        
        # window size - max_freq > k → shrink
        if right - left + 1 - max_freq > k:
            count[s[left]] -= 1
            left += 1
        
        result = max(result, right - left + 1)
    return result
```

### 5.4 Permutation in string
```python
def check_inclusion(s1, s2):
    if len(s1) > len(s2): return False
    need = Counter(s1)
    have = Counter(s2[:len(s1)])
    if have == need: return True
    
    for i in range(len(s1), len(s2)):
        have[s2[i]] += 1
        have[s2[i - len(s1)]] -= 1
        if have[s2[i - len(s1)]] == 0:
            del have[s2[i - len(s1)]]
        if have == need:
            return True
    return False
```

### 5.5 Subarray sum equals K (positive)
```python
def num_subarrays(arr, k):
    left = 0
    s = 0
    count = 0
    for right in range(len(arr)):
        s += arr[right]
        while s > k:
            s -= arr[left]
            left += 1
        if s == k:
            count += 1
            # NOTE: only works for positive numbers
    return count
```

---

## 6. Monotonic Deque — Max in window

### 문제 — 각 window 의 max
```
[1, 3, -1, -3, 5, 3, 6, 7], k=3
→ [3, 3, 5, 5, 6, 7]
```

### Sliding window + deque
```python
from collections import deque

def max_sliding_window(arr, k):
    dq = deque()       # indices, values decreasing
    result = []
    for i, v in enumerate(arr):
        # remove out of window
        while dq and dq[0] <= i - k:
            dq.popleft()
        # remove smaller from back
        while dq and arr[dq[-1]] < v:
            dq.pop()
        dq.append(i)
        if i >= k - 1:
            result.append(arr[dq[0]])
    return result
```

### 효과
- Naive O(N×k)
- Deque — O(N) (각 원소 1 번 push / pop)

---

## 7. Two pointers vs Sliding window

| | Two pointers | Sliding window |
| --- | --- | --- |
| **위치** | 어느 곳이든 | 연속 segment |
| **이동** | 양쪽 다 (보통) | 한 방향 |
| **state** | 작음 (sum / count) | window 안 정보 |

### 사실
- Sliding window 가 two pointers 의 **same direction 특수**
- 둘이 같이 분류되기도

---

## 8. 패턴 — 분기 표

| 문제 유형 | 패턴 |
| --- | --- |
| Fixed window of size k | Fixed sliding |
| Longest / shortest of condition | Variable sliding |
| Subarray sum (positive) | Variable + shrink |
| Subarray sum (any) | Prefix sum + hash |
| Count of distinct elements | Sliding + counter |
| Max in window | Monotonic deque |

---

## 9. 시간 복잡도

### O(N) — 각 원소 1 번 push, 1 번 pop
- 양쪽 포인터 각각 — N 번 movement max
- 총 2N = O(N)

### 공간 O(k) — window 안 + counter

---

## 10. 함정

### 함정 1 — Negative numbers
"Subarray sum K" — 음수 있으면 sliding window X. Prefix sum + hash.

### 함정 2 — Window 크기 변경 시 update
일부 문제 — window 한 단위 변경마다 — O(1) update.
못 하면 — naive O(N²).

### 함정 3 — Shrink 조건의 미묘
While 조건 — `>=` vs `>` — 1 차이.

### 함정 4 — Initial window
첫 k 원소 채우는 부분 — 별도 처리.

### 함정 5 — Counter 의 0 처리
딕셔너리 — 0 인 key 도 있음. `if count[c] == 0: del count[c]` 또는 `Counter` 그대로 비교.

### 함정 6 — Index vs value
일부 문제 — index 보존 (deque). 값만 보면 — 위치 정보 잃음.

---

## 11. 학습 자료

- LeetCode "Sliding Window" tag
- "Two Pointers" — Cracking the Coding Interview
- NeetCode — Sliding Window patterns
- "Algorithms" (CLRS) — Streaming

---

## 12. 관련

- [[arrays-and-strings]] — Hub
- [[two-pointers]] — 비슷한 패턴
- [[prefix-sum]] — 음수 / 임의 합
- [[../stacks-and-queues/stacks-and-queues]] — Monotonic deque
