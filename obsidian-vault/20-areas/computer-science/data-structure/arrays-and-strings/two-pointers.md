---
title: "Two Pointers — 투 포인터 패턴"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T11:10:00+09:00
tags:
  - data-structure
  - pattern
  - two-pointers
---

# Two Pointers — 투 포인터 패턴

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Opposite / Same / Fast-Slow |

**[[arrays-and-strings|↑ Arrays & Strings]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

배열 / 문자열의 **두 인덱스를 동시에 움직여** O(N) 으로 푸는 패턴. 이중 loop O(N²) 를 O(N) 으로.

---

## 2. 3 가지 변형

### 2.1 Opposite (양 끝)
```
[1, 2, 3, 4, 5]
 ↑           ↑
left       right
```

### 2.2 Same direction (한쪽)
```
[1, 2, 3, 4, 5]
 ↑↑
slow fast
```

### 2.3 Fast & Slow (속도 차)
```
[1, 2, 3, 4, 5]
 ↑     ↑
slow  fast (2 steps)
```

---

## 3. 패턴 1 — Opposite (Two-Sum sorted, palindrome)

### Two Sum (sorted)
```python
def two_sum_sorted(arr, target):
    left, right = 0, len(arr) - 1
    while left < right:
        s = arr[left] + arr[right]
        if s == target:
            return [left, right]
        elif s < target:
            left += 1
        else:
            right -= 1
    return None
```

### Palindrome check
```python
def is_palindrome(s):
    left, right = 0, len(s) - 1
    while left < right:
        if s[left] != s[right]:
            return False
        left += 1
        right -= 1
    return True
```

### Container with most water
```python
def max_area(heights):
    left, right = 0, len(heights) - 1
    max_a = 0
    while left < right:
        area = min(heights[left], heights[right]) * (right - left)
        max_a = max(max_a, area)
        if heights[left] < heights[right]:
            left += 1
        else:
            right -= 1
    return max_a
```

### Reverse string
```python
def reverse(arr):
    left, right = 0, len(arr) - 1
    while left < right:
        arr[left], arr[right] = arr[right], arr[left]
        left += 1
        right -= 1
```

---

## 4. 패턴 2 — Same direction (slow / fast)

### Remove duplicates from sorted
```python
def remove_duplicates(arr):
    if not arr: return 0
    slow = 0
    for fast in range(1, len(arr)):
        if arr[fast] != arr[slow]:
            slow += 1
            arr[slow] = arr[fast]
    return slow + 1
```

### Move zeros to end
```python
def move_zeros(arr):
    slow = 0
    for fast in range(len(arr)):
        if arr[fast] != 0:
            arr[slow], arr[fast] = arr[fast], arr[slow]
            slow += 1
```

### 0/1/2 partition (Dutch national flag)
```python
def sort_colors(arr):
    low, mid, high = 0, 0, len(arr) - 1
    while mid <= high:
        if arr[mid] == 0:
            arr[low], arr[mid] = arr[mid], arr[low]
            low += 1
            mid += 1
        elif arr[mid] == 1:
            mid += 1
        else:    # arr[mid] == 2
            arr[mid], arr[high] = arr[high], arr[mid]
            high -= 1
```

---

## 5. 패턴 3 — Fast & Slow (속도 차)

### Linked list cycle detection (Floyd's)
```python
def has_cycle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow == fast:
            return True
    return False
```

### Find middle of linked list
```python
def middle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
    return slow
```

### Find duplicate (Floyd's on array)
```python
def find_duplicate(arr):
    slow = arr[0]
    fast = arr[0]
    
    # Phase 1: cycle detection
    while True:
        slow = arr[slow]
        fast = arr[arr[fast]]
        if slow == fast: break
    
    # Phase 2: find entry
    slow = arr[0]
    while slow != fast:
        slow = arr[slow]
        fast = arr[fast]
    return slow
```

---

## 6. 패턴 4 — 다중 array merge

### Merge sorted arrays
```python
def merge(a, b):
    result = []
    i = j = 0
    while i < len(a) and j < len(b):
        if a[i] < b[j]:
            result.append(a[i])
            i += 1
        else:
            result.append(b[j])
            j += 1
    result.extend(a[i:])
    result.extend(b[j:])
    return result
```

### Merge intervals
```python
def merge_intervals(intervals):
    intervals.sort()
    result = [intervals[0]]
    for start, end in intervals[1:]:
        if start <= result[-1][1]:
            result[-1][1] = max(result[-1][1], end)
        else:
            result.append([start, end])
    return result
```

---

## 7. 패턴 5 — 3-Sum / N-Sum

### 3 Sum
```python
def three_sum(arr):
    arr.sort()
    result = []
    for i in range(len(arr) - 2):
        if i > 0 and arr[i] == arr[i-1]: continue    # dedup
        left, right = i + 1, len(arr) - 1
        while left < right:
            s = arr[i] + arr[left] + arr[right]
            if s == 0:
                result.append([arr[i], arr[left], arr[right]])
                while left < right and arr[left] == arr[left+1]: left += 1
                while left < right and arr[right] == arr[right-1]: right -= 1
                left += 1
                right -= 1
            elif s < 0:
                left += 1
            else:
                right -= 1
    return result
```

---

## 8. 조건 — 언제 투 포인터?

### O 사용 가능
- **정렬된** 배열 (대부분)
- 두 방향 단조 — 한쪽 ↑ 한쪽 ↓
- 합 / 차 / 곱 — 단조 변화

### X 안 됨
- 정렬 안 됨 — 정렬 후 또는 hash
- 임의 두 쌍 검색 — O(N²) 이중 loop

---

## 9. 시간 / 공간

| | 시간 | 공간 |
| --- | --- | --- |
| Two pointers | O(N) | O(1) |
| Nested loop (naive) | O(N²) | O(1) |
| Hash 활용 | O(N) | O(N) |

→ 정렬된 — 투 포인터가 best.
→ 정렬 안 됨 — hash 또는 정렬 후 투 포인터 (O(N log N)).

---

## 10. 함정

### 함정 1 — Infinite loop
조건 안 맞으면 — left/right 안 움직임. 항상 한쪽 update.

### 함정 2 — Index out of bounds
`fast.next.next` — `fast.next` 가 None 인지 먼저.

### 함정 3 — Dedup 누락
같은 값 — 중복 결과. While dedup 또는 set.

### 함정 4 — 정렬 안 함
정렬된 가정 — 정렬 안 한 입력은 fail. 정렬 X 면 hash.

### 함정 5 — Left/right cross
`left < right` vs `left <= right` — 문제 따라.

### 함정 6 — Modification 중 iteration
Index 가리키는 위치 변경 — 인덱스 재조정.

---

## 11. 흔한 LeetCode

| 문제 | 패턴 |
| --- | --- |
| Two Sum II | Opposite |
| Valid Palindrome | Opposite |
| Container With Most Water | Opposite |
| 3 Sum | Opposite (nested) |
| Remove Duplicates | Same direction |
| Move Zeroes | Same direction |
| Sort Colors | 3-way (Dutch flag) |
| Linked List Cycle | Fast-Slow |
| Find Duplicate | Fast-Slow on array |
| Merge Sorted Array | Two arrays |

---

## 12. 학습 자료

- LeetCode "Two Pointers" tag
- "Cracking the Coding Interview" — array chapter
- NeetCode — Two Pointers patterns

---

## 13. 관련

- [[arrays-and-strings]] — Hub
- [[sliding-window]] — 비슷한 패턴
- [[../linked-lists/linked-lists]] — Floyd cycle
