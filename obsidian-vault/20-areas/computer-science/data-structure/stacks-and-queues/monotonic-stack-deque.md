---
title: "Monotonic Stack / Deque — 패턴"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T13:15:00+09:00
tags:
  - data-structure
  - monotonic
  - pattern
---

# Monotonic Stack / Deque

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 단조 / next-greater / 윈도우 max |

**[[stacks-and-queues|↑ Stacks & Queues]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

스택 / 데큐 안의 **원소들이 단조** (증가 / 감소) 한 상태 유지. O(N) 으로 "next greater" / "previous smaller" / "window max" 등 해결.

---

## 2. 핵심 아이디어

```
원소 추가 시 — 단조 깨는 원소 모두 pop.
각 원소 — push 1 번 / pop 1 번 → O(N).
```

### 단조 감소 stack (값이 위에서 아래로 ↑)
```
push x:
    while stack.top() <= x:
        pop()
    push(x)
```

### 단조 증가 stack
```
push x:
    while stack.top() >= x:
        pop()
    push(x)
```

---

## 3. 패턴 1 — Next Greater Element

### 문제
- 각 원소의 오른쪽 첫 번째 더 큰 원소

```python
def next_greater(arr):
    n = len(arr)
    result = [-1] * n
    stack = []        # indices, decreasing
    for i in range(n):
        while stack and arr[stack[-1]] < arr[i]:
            j = stack.pop()
            result[j] = arr[i]
        stack.append(i)
    return result
```

### 변형
- Next smaller — 부호 반대
- Previous greater / smaller — 역방향 iteration

### 예
```
arr =     [2, 1, 2, 4, 3, 1]
result =  [4, 2, 4,-1,-1,-1]
```

---

## 4. 패턴 2 — Daily Temperatures

### 문제
- 각 day 의 다음 더 따뜻한 day 까지 며칠?

```python
def daily_temperatures(T):
    n = len(T)
    result = [0] * n
    stack = []
    for i in range(n):
        while stack and T[stack[-1]] < T[i]:
            j = stack.pop()
            result[j] = i - j
        stack.append(i)
    return result
```

---

## 5. 패턴 3 — Largest Rectangle in Histogram

### 문제
- 막대 그래프의 가장 큰 직사각형

```python
def largest_rectangle(heights):
    stack = []          # indices, increasing heights
    max_area = 0
    heights.append(0)   # sentinel
    for i, h in enumerate(heights):
        while stack and heights[stack[-1]] > h:
            top = stack.pop()
            left = -1 if not stack else stack[-1]
            width = i - left - 1
            max_area = max(max_area, heights[top] * width)
        stack.append(i)
    return max_area
```

### 응용 — Maximal Rectangle in Binary Matrix
- 각 row 별 histogram → largest rectangle
- O(M×N)

---

## 6. 패턴 4 — Trap Rain Water

### 문제
- 막대 사이의 빗물

```python
def trap(height):
    stack = []
    water = 0
    for i, h in enumerate(height):
        while stack and height[stack[-1]] < h:
            mid = stack.pop()
            if not stack: break
            left = stack[-1]
            width = i - left - 1
            bounded_h = min(height[left], h) - height[mid]
            water += width * bounded_h
        stack.append(i)
    return water
```

---

## 7. 패턴 5 — Sliding Window Max (Deque)

```python
from collections import deque

def max_sliding_window(arr, k):
    dq = deque()          # indices, decreasing values
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

### 시간
- 각 원소 — push 1, pop 1 → O(N)

---

## 8. 패턴 6 — Sum of Subarray Minimums

### 문제
- 모든 subarray 의 min 의 합

```python
def sum_subarray_mins(arr):
    n = len(arr)
    MOD = 10**9 + 7
    
    # left[i] = i 보다 왼쪽의 첫 번째 smaller index
    left = [-1] * n
    stack = []
    for i in range(n):
        while stack and arr[stack[-1]] > arr[i]:
            stack.pop()
        left[i] = stack[-1] if stack else -1
        stack.append(i)
    
    # right[i] = i 보다 오른쪽의 첫 번째 smaller (or equal)
    right = [n] * n
    stack = []
    for i in range(n):
        while stack and arr[stack[-1]] >= arr[i]:
            right[stack.pop()] = i
        stack.append(i)
    
    return sum(arr[i] * (i - left[i]) * (right[i] - i) for i in range(n)) % MOD
```

---

## 9. 단조 stack — 분류

| 단조 | Stack 위 | 사용 |
| --- | --- | --- |
| Decreasing | 값 큰 ← 작은 | Next greater, daily temp |
| Increasing | 값 작은 ← 큰 | Largest rectangle, prev smaller |

### Tip
- 값이 단조 깨질 때 — 깨진 원소가 pop 의 boundary
- pop 의 결과 — "이 원소의 next greater / smaller" 알 수 있음

---

## 10. 단조 deque vs 단조 stack

| | Stack | Deque |
| --- | --- | --- |
| **사용** | one-pass scan | Sliding window |
| **양 끝** | 한쪽만 | 양쪽 (window 벗어남 → front pop) |

---

## 11. 함정

### 함정 1 — 등호 처리
`< vs <=` — 문제에 따라. Sum of mins — 한쪽 strict, 한쪽 non-strict (중복 처리).

### 함정 2 — Sentinel
배열 끝 0 추가 — 마지막 단조 stack 비우기. 코드 단순화.

### 함정 3 — Index vs value
저장 — index (위치 알아야), 비교 — arr[index].

### 함정 4 — 매번 stack 새로 만들기
한 pass 에 한 stack. 함수 호출 시 reset.

### 함정 5 — Overflow
큰 N — long / 모듈러.

### 함정 6 — Deque 의 front pop
Sliding — index <= i - k 인지 확인. 인덱스 위주.

---

## 12. 흔한 LeetCode

| 문제 | 패턴 |
| --- | --- |
| Daily Temperatures | Next greater |
| Next Greater Element I/II | Next greater |
| Trapping Rain Water | Monotonic stack |
| Largest Rectangle in Histogram | Monotonic stack |
| Maximal Rectangle | Histogram per row |
| Sum of Subarray Minimums | Prev/next smaller |
| Sliding Window Maximum | Monotonic deque |
| Constrained Subset Sum | Monotonic deque + DP |
| Remove K Digits | Monotonic stack |
| 132 Pattern | Monotonic stack |

---

## 13. 학습 자료

- LeetCode "Monotonic Stack" tag
- "Algorithm Patterns" — NeetCode
- Competitive Programming Handbook

---

## 14. 관련

- [[stacks-and-queues]] — Hub
- [[stack]] / [[deque]]
- [[../arrays-and-strings/sliding-window]] — Sliding window 변형
