---
title: "배열 / 문자열 (Arrays & Strings)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T05:30:00+09:00
tags:
  - data-structure
  - array
  - string
---

# 배열 / 문자열 (Arrays & Strings)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 수준 + 11 언어 + 실습 |

**[[../data-structure|↑ data-structure]]** · **[[../../computer-science|↑↑ computer-science]]**

---

## 1. 한 줄 정의

**연속된 메모리 공간에 같은 타입의 값을 나열**한 가장 기본 자료구조.
인덱스로 O(1) 접근. 모든 다른 자료구조의 토대.

---

## 2. 언제 쓰는가? — 시그널 인식

### 배열이 최적인 경우

- **인덱스로 무작위 접근이 자주** — O(1)
- **순차 순회가 자주** — 캐시 친화적 (cache-friendly)
- **크기가 고정 또는 예측 가능**
- **수치 계산 / 행렬 / 영상 처리** — SIMD 가능

### 배열을 피해야 할 경우

- **앞쪽에 빈번한 삽입·삭제** — `O(N)` shift 필요 → Linked List / Deque
- **크기를 모름 + 거의 안 늘어남** → 메모리 낭비. Dynamic Array (`ArrayList` / `Vec`) 가 해결.
- **키로 검색** — Hash Table 이 O(1).

### 정적 배열 vs 동적 배열 (Dynamic Array)

| 차원 | 정적 배열 | 동적 배열 (Vec / ArrayList) |
| --- | --- | --- |
| 크기 | 컴파일 시 결정 | 런타임 가변 |
| 메모리 | 스택 (작은 것) / 힙 | 힙 |
| 확장 | ❌ | 2배씩 (amortized O(1)) |
| Python `list` | — | 동적 배열 |
| Java `int[]` | 정적 | `ArrayList` 는 동적 |
| C `int[10]` | 정적 | `malloc` 으로 직접 |
| Go `[10]int` | 정적 | `[]int` (slice) 동적 |

---

## 3. 핵심 직관 — 메모리 레이아웃

```
배열 a = [10, 20, 30, 40, 50]
주소:    0x100  0x104  0x108  0x10C  0x110

a[2] 접근:
  base_address + (2 × sizeof(int))
  = 0x100 + 8
  = 0x108
  → 10진수 30
```

**한 번의 곱셈 + 덧셈**으로 어떤 위치든 O(1). 캐시 라인 (보통 64 byte) 안의
인접 원소는 한 번의 memory load 로 가져옴 → 순차 접근 매우 빠름.

### 다차원 배열의 메모리 레이아웃

```
2D 배열 m[3][4]:

Row-major (C, C++, Python NumPy 기본):
  m[0][0], m[0][1], m[0][2], m[0][3],
  m[1][0], m[1][1], ..., m[2][3]
  → 같은 행 원소가 인접

Column-major (Fortran, MATLAB):
  m[0][0], m[1][0], m[2][0],
  m[0][1], m[1][1], ...
  → 같은 열 원소가 인접

성능 차이:
  for i in range(N):
      for j in range(M):
          x = m[i][j]   # row-major: 캐시 친화적 (빠름)
                          # column-major: 캐시 미스 (느림)
```

---

## 4. 동작 원리 — 핵심 연산

### 정적 배열 (예: C / Java `int[]`)

```
int a[5];                # 메모리 5 슬롯 할당, 인접
a[0] = 10;               # O(1)
int x = a[2];            # O(1) (인덱스 계산만)
a[3] = a[3] + 1;         # O(1)
크기 증가? → ❌ 불가
```

### 동적 배열 (예: Python `list`, C++ `vector`)

```
초기: capacity=4, size=0, data = [_, _, _, _]

append(10): size=1, [10, _, _, _]
append(20): size=2, [10, 20, _, _]
append(30): size=3, [10, 20, 30, _]
append(40): size=4, [10, 20, 30, 40]
append(50): size > capacity → 재할당!
  새 capacity = 8 (2배)
  복사 + 새 원소: [10, 20, 30, 40, 50, _, _, _]
```

**Amortized O(1)**: 재할당이 N 번에 한 번씩 (2배 정책) 일어나고 그 비용은 O(N) → N 번 호출 평균 O(1).

### 중간 삽입 / 삭제 — O(N)

```
a = [10, 20, 30, 40, 50]
insert(2, 99) → [10, 20, 99, 30, 40, 50]
  → 30, 40, 50 을 한 칸씩 오른쪽 shift

delete(2) → [10, 20, 30, 40]    (원래 a 에서 30 삭제)
  → 40, 50 을 한 칸씩 왼쪽 shift
```

---

## 5. 복잡도 종합

| 연산 | 정적 배열 | 동적 배열 | 비고 |
| --- | --- | --- | --- |
| `a[i]` 접근 | O(1) | O(1) | 인덱스 계산만 |
| `a[i] = x` 갱신 | O(1) | O(1) | |
| 끝에 추가 (push) | ❌ | **O(1) amortized** | 재할당 시 O(N) |
| 끝에서 제거 (pop) | ❌ | O(1) | |
| 앞에 추가 / 제거 | O(N) | O(N) | shift |
| 중간 삽입 / 삭제 | O(N) | O(N) | shift |
| 검색 (값) | O(N) | O(N) | 정렬되어 있으면 O(log N) 이진탐색 |
| 정렬 | O(N log N) | O(N log N) | |
| 공간 | O(N) | O(N) | |

---

## 6. 문자열 = 문자의 배열

문자열은 **immutable** vs **mutable** 차이가 언어마다 다름:

| 언어 | 기본 문자열 | mutable? | concat 비용 |
| --- | --- | --- | --- |
| Python | `str` | ❌ immutable | O(N+M) — 새 객체 |
| Java | `String` | ❌ | O(N+M) — `StringBuilder` 사용 권장 |
| C++ | `std::string` | ✅ | O(1) amortized (append) |
| C | `char[]` | ✅ | 직접 관리 |
| JavaScript | `string` | ❌ | O(N+M) |
| Kotlin / Swift | 기본 immutable | — | StringBuilder / 인라인 |
| Go | `string` | ❌ | `strings.Builder` 권장 |
| Rust | `String` | ✅ | O(1) amortized |

**핵심**: immutable 언어 (Python / Java / JS) 에서 `s += c` 를 N 번 하면 O(N²). 항상 **빌더 / list 누적 후 join**.

```python
# 잘못 - O(N²)
s = ""
for c in chars: s += c

# 옳음 - O(N)
s = "".join(chars)
```

---

## 7. 언어별 구현 (11 언어) — 동적 배열 + 문자열

### Python 3

```python
# 동적 배열 = list
a = [1, 2, 3]
a.append(4)              # O(1) amortized → [1, 2, 3, 4]
a.insert(0, 0)           # O(N) → [0, 1, 2, 3, 4]
x = a[2]                 # O(1)
a[2] = 99                # O(1)
del a[1]                 # O(N)
a.sort()                 # O(N log N)
# 리스트 컴프리헨션
squared = [x*x for x in a if x > 0]

# 문자열
s = "hello"
parts = s.split('l')      # ['he', '', 'o']
joined = ''.join(parts)   # 'heo'
sub = s[1:4]              # 'ell'

# StringBuilder 패턴
chars = []
for i in range(1000):
    chars.append(str(i))
result = ''.join(chars)   # O(N)

# 2D 배열 (주의: 얕은 복사)
matrix = [[0]*N for _ in range(M)]   # 올바름
matrix_wrong = [[0]*N] * M           # 잘못 — 같은 리스트 M 번 참조
```

### Java

```java
import java.util.*;
public class ArrayDemo {
    public static void main(String[] args) {
        // 정적 배열
        int[] a = {1, 2, 3};
        int x = a[0];                        // O(1)
        Arrays.sort(a);                       // O(N log N)
        int[] copy = a.clone();
        int[] sub = Arrays.copyOfRange(a, 0, 2);

        // 동적 배열
        ArrayList<Integer> list = new ArrayList<>();
        list.add(1); list.add(2); list.add(3);   // O(1) amortized
        list.add(0, 99);                          // O(N)
        list.remove(0);                            // O(N)
        Integer y = list.get(0);                   // O(1)

        // 문자열 (immutable)
        String s = "hello";
        String sub2 = s.substring(1, 4);           // "ell"
        char c = s.charAt(2);                       // 'l'

        // StringBuilder (가변)
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < 1000; i++) sb.append(i);
        String result = sb.toString();

        // 2D 배열
        int[][] matrix = new int[3][4];
        // 또는 동적
        List<List<Integer>> grid = new ArrayList<>();
    }
}
```

### C++

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    // 정적 배열
    int a[5] = {1, 2, 3, 4, 5};
    // 또는 std::array
    array<int, 5> arr = {1, 2, 3, 4, 5};

    // 동적 배열 - vector
    vector<int> v = {1, 2, 3};
    v.push_back(4);                          // O(1) amortized
    v.insert(v.begin(), 0);                  // O(N)
    v.erase(v.begin() + 2);                  // O(N)
    sort(v.begin(), v.end());

    // 문자열 (mutable)
    string s = "hello";
    s += " world";                            // O(1) amortized
    string sub = s.substr(1, 3);              // "ell"

    // 2D 벡터
    vector<vector<int>> matrix(N, vector<int>(M, 0));
}
```

### C

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main() {
    // 정적 배열
    int a[5] = {1, 2, 3, 4, 5};

    // 동적 배열 (직접 관리)
    int *dyn = malloc(10 * sizeof(int));
    for (int i = 0; i < 10; i++) dyn[i] = i;

    // 크기 늘리기
    dyn = realloc(dyn, 20 * sizeof(int));
    free(dyn);

    // 문자열
    char s[100] = "hello";
    strcat(s, " world");                      // 주의: 버퍼 오버플로
    char *substr = strstr(s, "world");        // 부분 문자열 찾기

    // 동적 문자열
    char *str = malloc(100);
    snprintf(str, 100, "hello %s", "world");
    free(str);

    // 2D 배열
    int matrix[3][4];
    // 또는 동적
    int **m = malloc(N * sizeof(int*));
    for (int i = 0; i < N; i++) m[i] = calloc(M, sizeof(int));
}
```

### JavaScript (Node.js)

```javascript
// Array (사실상 동적 배열)
const a = [1, 2, 3];
a.push(4);                              // O(1)
a.unshift(0);                           // O(N)
a.splice(2, 1);                         // O(N) - 삭제
a.splice(2, 0, 99);                     // O(N) - 삽입
const x = a[2];                         // O(1)
a.sort((x, y) => x - y);                // O(N log N)
const filtered = a.filter(v => v > 0);
const mapped = a.map(v => v * 2);

// 문자열 (immutable)
const s = "hello";
const sub = s.substring(1, 4);           // "ell"
const joined = ["a","b","c"].join('-');  // "a-b-c"

// 빌더 패턴 - 배열에 push 후 join
const parts = [];
for (let i = 0; i < 1000; i++) parts.push(i);
const result = parts.join('');

// 2D
const matrix = Array.from({length: N}, () => new Array(M).fill(0));
```

### TypeScript

```typescript
const a: number[] = [1, 2, 3];
a.push(4);

// 타입 안전한 2D
const matrix: number[][] = Array.from({length: N}, () => new Array(M).fill(0));

// 인터페이스 + 배열
interface User { name: string; age: number; }
const users: User[] = [{name: "Alice", age: 30}];

// 튜플
const point: [number, number] = [1, 2];

// readonly
const frozen: readonly number[] = [1, 2, 3];   // 수정 불가
```

### Kotlin

```kotlin
fun main() {
    // 정적 배열
    val a = intArrayOf(1, 2, 3, 4, 5)
    val x = a[0]

    // 동적
    val list = mutableListOf(1, 2, 3)
    list.add(4)
    list.add(0, 99)
    list.removeAt(0)
    list.sort()

    // 문자열 (immutable)
    val s = "hello"
    val sub = s.substring(1, 4)              // "ell"

    // StringBuilder
    val sb = StringBuilder()
    repeat(1000) { sb.append(it) }
    val result = sb.toString()

    // 2D
    val matrix = Array(N) { IntArray(M) }
}
```

### Swift

```swift
// 동적 배열
var a = [1, 2, 3]
a.append(4)
a.insert(0, at: 0)
a.remove(at: 2)
let x = a[2]
a.sort()

// 문자열 (immutable by default)
let s = "hello"
let startIdx = s.index(s.startIndex, offsetBy: 1)
let endIdx = s.index(s.startIndex, offsetBy: 4)
let sub = String(s[startIdx..<endIdx])     // "ell"

// 2D
var matrix = Array(repeating: Array(repeating: 0, count: N), count: M)
```

### Go

```go
package main

import (
    "fmt"
    "sort"
    "strings"
)

func main() {
    // 정적 배열
    var arr [5]int = [5]int{1, 2, 3, 4, 5}

    // 슬라이스 (동적)
    s := []int{1, 2, 3}
    s = append(s, 4)                          // O(1) amortized
    s = append([]int{0}, s...)                // O(N) — 앞에 삽입
    sort.Ints(s)

    // 문자열 (immutable, byte slice)
    str := "hello"
    sub := str[1:4]                           // "ell"

    // StringBuilder
    var sb strings.Builder
    for i := 0; i < 1000; i++ {
        fmt.Fprintf(&sb, "%d", i)
    }
    result := sb.String()

    // 2D 슬라이스
    matrix := make([][]int, M)
    for i := range matrix {
        matrix[i] = make([]int, N)
    }
}
```

### Rust

```rust
fn main() {
    // 정적 배열
    let a: [i32; 5] = [1, 2, 3, 4, 5];

    // 동적 Vec
    let mut v: Vec<i32> = vec![1, 2, 3];
    v.push(4);
    v.insert(0, 0);
    v.remove(2);
    v.sort();

    // 문자열
    let s = String::from("hello");
    let sub = &s[1..4];                       // "ell"
    
    // String 빌더 (Vec<char> 또는 String)
    let mut s2 = String::new();
    for i in 0..1000 {
        s2.push_str(&i.to_string());
    }

    // 2D
    let matrix: Vec<Vec<i32>> = vec![vec![0; N]; M];

    // 슬라이스 (참조)
    let slice: &[i32] = &v[1..3];
}
```

### C#

```csharp
using System;
using System.Collections.Generic;
using System.Text;

public class ArrayDemo {
    public static void Main() {
        // 정적 배열
        int[] a = {1, 2, 3, 4, 5};
        Array.Sort(a);

        // 동적 List
        var list = new List<int> {1, 2, 3};
        list.Add(4);
        list.Insert(0, 99);
        list.RemoveAt(0);
        int x = list[2];

        // 문자열 (immutable)
        string s = "hello";
        string sub = s.Substring(1, 3);        // "ell"

        // StringBuilder
        var sb = new StringBuilder();
        for (int i = 0; i < 1000; i++) sb.Append(i);
        string result = sb.ToString();

        // 2D
        int[,] matrix = new int[N, M];
        // 또는 jagged
        int[][] jagged = new int[N][];
        for (int i = 0; i < N; i++) jagged[i] = new int[M];
    }
}
```

---

## 8. 핵심 패턴 / 변형

### (a) 투 포인터 (Two Pointers)

정렬된 배열 / 부분 합 / 회문 검사에 활용.

```python
# 두 수의 합 (정렬된 배열에서)
def two_sum(arr, target):
    lo, hi = 0, len(arr) - 1
    while lo < hi:
        s = arr[lo] + arr[hi]
        if s == target: return (lo, hi)
        elif s < target: lo += 1
        else: hi -= 1
    return None
```

### (b) 슬라이딩 윈도우

연속된 부분 배열의 합 / 최댓값 / 길이.

```python
# 길이 K 부분 배열의 최대 합
def max_sum(arr, k):
    window_sum = sum(arr[:k])
    max_sum = window_sum
    for i in range(k, len(arr)):
        window_sum += arr[i] - arr[i-k]   # O(1) 갱신
        max_sum = max(max_sum, window_sum)
    return max_sum
```

### (c) 누적합 (Prefix Sum)

구간 합 쿼리 O(1).

```python
def build_prefix(arr):
    prefix = [0]
    for x in arr: prefix.append(prefix[-1] + x)
    return prefix
    # range_sum(l, r) = prefix[r+1] - prefix[l]
```

### (d) 카데인 알고리즘 (Kadane)

최대 부분 배열 합 O(N).

```python
def max_subarray(arr):
    best = cur = arr[0]
    for x in arr[1:]:
        cur = max(x, cur + x)
        best = max(best, cur)
    return best
```

### (e) 반전 / 회전

```python
arr.reverse()                  # in-place 반전 O(N)
arr = arr[::-1]                # 새 배열 반전

# 왼쪽으로 k 번 회전
def rotate(arr, k):
    k %= len(arr)
    return arr[k:] + arr[:k]
```

### (f) 듀얼 패스 (Two-Pass)

배열을 두 번 순회 — 누적합 / 좌·우 최대값 등.

---

## 9. 함정 / 안티패턴

### 함정 1 — Python `list * N` 의 얕은 복사

```python
# 잘못 - 같은 리스트 N 번 참조
matrix = [[0] * M] * N
matrix[0][0] = 1
# matrix[1][0] 도 1 이 됨!

# 옳음
matrix = [[0] * M for _ in range(N)]
```

### 함정 2 — 문자열 누적 += 의 O(N²)

immutable 문자열 언어 (Python / Java / JS) 에서 `s += c` 반복은 O(N²).
**빌더 / list 누적 + join 사용**.

### 함정 3 — 인덱스 Out-of-Bounds

```python
# 잘못
for i in range(len(arr)):
    if arr[i+1] > arr[i]: ...    # i+1 = len(arr) 에서 IndexError

# 옳음
for i in range(len(arr) - 1):
    if arr[i+1] > arr[i]: ...
```

### 함정 4 — 0-based vs 1-based

문제마다 다름. 입력 받자마자 통일. 일반적으로 코드 0-based + 출력 시 +1.

### 함정 5 — 부동소수점 배열 비교

`a == b` 직접 비교 X. `abs(a - b) < eps` 사용.

### 함정 6 — 큰 배열에 `in` 연산

```python
# 잘못 - O(N) per check
if x in big_list: ...

# 옳음 - O(1)
big_set = set(big_list)
if x in big_set: ...
```

### 함정 7 — Java `int[]` 의 `Arrays.asList`

```java
Integer[] arr = {1, 2, 3};
List<Integer> list = Arrays.asList(arr);   // immutable view!
list.add(4);   // UnsupportedOperationException
```

`new ArrayList<>(Arrays.asList(arr))` 사용.

### 함정 8 — JavaScript Array sort 의 문자열 비교

```javascript
[10, 2, 1].sort();        // ['1', '10', '2'] ← 문자열 비교!
[10, 2, 1].sort((a, b) => a - b);   // [1, 2, 10]
```

### 함정 9 — 동적 배열 size vs capacity

`v.size()` 는 실제 원소 수, `v.capacity()` 는 할당된 공간. 헷갈리지 말 것.

### 함정 10 — Go slice 의 underlying array 공유

```go
s1 := []int{1, 2, 3, 4, 5}
s2 := s1[1:3]
s2[0] = 99   // s1 도 영향 받음 → s1 = [1, 99, 3, 4, 5]
```

복사 원하면 `copy(s2, s1[1:3])` 또는 `append([]int{}, s1[1:3]...)`.

---

## 10. 실전 풀이 절차

1. **자료구조 확정** — 정적 / 동적 / 해시 / 정렬된 배열 중 무엇?
2. **인덱스 0/1 통일** — 입력 변환.
3. **메모리 충분?** — N=10^6 정수 → 4MB. OK. N=10^4 × 10^4 → 400MB 메모리 초과.
4. **연산 복잡도** — O(N²) 가 N=10^5 면 TLE. 패턴 (투 포인터 / 슬라이딩 / 누적합) 적용.
5. **엣지** — 빈 배열 / N=1 / 음수 / 중복 / 모두 같은 값.

---

## 11. 대표 문제

### Q1. 두 수의 합 (LeetCode 1)

```python
# Hash Table 활용 - O(N)
def two_sum(nums, target):
    seen = {}
    for i, x in enumerate(nums):
        if target - x in seen: return [seen[target-x], i]
        seen[x] = i
```

### Q2. 최대 부분 배열 합 (LeetCode 53) — Kadane

```python
def max_subarray(nums):
    best = cur = nums[0]
    for x in nums[1:]:
        cur = max(x, cur + x)
        best = max(best, cur)
    return best
```

### Q3. 회전된 정렬 배열 검색 (LeetCode 33)

```python
def search(nums, target):
    lo, hi = 0, len(nums) - 1
    while lo <= hi:
        mid = (lo + hi) // 2
        if nums[mid] == target: return mid
        if nums[lo] <= nums[mid]:        # 왼쪽이 정렬됨
            if nums[lo] <= target < nums[mid]: hi = mid - 1
            else: lo = mid + 1
        else:                              # 오른쪽이 정렬됨
            if nums[mid] < target <= nums[hi]: lo = mid + 1
            else: hi = mid - 1
    return -1
```

### Q4. 가장 긴 부분 회문 (LeetCode 5)

```python
def longest_palindrome(s):
    if not s: return ""
    start = end = 0
    for i in range(len(s)):
        l1 = expand(s, i, i)       # 홀수 길이
        l2 = expand(s, i, i+1)     # 짝수 길이
        l = max(l1, l2)
        if l > end - start:
            start = i - (l - 1) // 2
            end = i + l // 2
    return s[start:end+1]

def expand(s, l, r):
    while l >= 0 and r < len(s) and s[l] == s[r]:
        l -= 1; r += 1
    return r - l - 1
```

### Q5. 회문 (LeetCode 125)

```python
def is_palindrome(s):
    s = ''.join(c.lower() for c in s if c.isalnum())
    return s == s[::-1]
```

### Q6. 부분 배열 합 최댓값 (슬라이딩 윈도우, LeetCode 643)

```python
def find_max_average(nums, k):
    window_sum = sum(nums[:k])
    max_sum = window_sum
    for i in range(k, len(nums)):
        window_sum += nums[i] - nums[i-k]
        max_sum = max(max_sum, window_sum)
    return max_sum / k
```

---

## 12. 연습 문제 (난이도별)

### 입문
- [백준 10818 최소 최대](https://www.acmicpc.net/problem/10818)
- [백준 1546 평균](https://www.acmicpc.net/problem/1546)
- [LeetCode 1 Two Sum](https://leetcode.com/problems/two-sum/)
- [LeetCode 88 Merge Sorted Array](https://leetcode.com/problems/merge-sorted-array/)

### 중급
- [백준 11659 구간 합 구하기 4](https://www.acmicpc.net/problem/11659) — 누적합
- [백준 2003 수들의 합 2](https://www.acmicpc.net/problem/2003) — 슬라이딩
- [LeetCode 53 Maximum Subarray](https://leetcode.com/problems/maximum-subarray/) — Kadane
- [LeetCode 121 Best Time to Buy/Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/)

### 고급
- [LeetCode 42 Trapping Rain Water](https://leetcode.com/problems/trapping-rain-water/)
- [LeetCode 76 Minimum Window Substring](https://leetcode.com/problems/minimum-window-substring/)
- [LeetCode 239 Sliding Window Maximum](https://leetcode.com/problems/sliding-window-maximum/)

---

## 13. 실습 코드 — 직접 동적 배열 구현 (Python)

```python
class DynamicArray:
    def __init__(self):
        self._capacity = 4
        self._size = 0
        self._data = [None] * self._capacity

    def __len__(self): return self._size

    def __getitem__(self, i):
        if not 0 <= i < self._size: raise IndexError
        return self._data[i]

    def __setitem__(self, i, value):
        if not 0 <= i < self._size: raise IndexError
        self._data[i] = value

    def append(self, value):
        if self._size == self._capacity:
            self._resize(2 * self._capacity)
        self._data[self._size] = value
        self._size += 1

    def _resize(self, new_capacity):
        new_data = [None] * new_capacity
        for i in range(self._size):
            new_data[i] = self._data[i]
        self._data = new_data
        self._capacity = new_capacity

    def insert(self, i, value):
        if self._size == self._capacity: self._resize(2 * self._capacity)
        for j in range(self._size, i, -1):
            self._data[j] = self._data[j-1]
        self._data[i] = value
        self._size += 1

    def remove(self, i):
        for j in range(i, self._size - 1):
            self._data[j] = self._data[j+1]
        self._size -= 1

# 테스트
arr = DynamicArray()
for x in range(10): arr.append(x)
print([arr[i] for i in range(len(arr))])   # [0..9]
arr.insert(0, -1)
arr.remove(5)
```

---

## 14. 학습 자료

### 책
- Introduction to Algorithms (CLRS) Ch.10
- The Algorithm Design Manual (Skiena) Ch.3
- 이코테 Appendix A (Python 문법) — `list` / `string` 기초

### 강의
- [MIT 6.006 — Sequences](https://ocw.mit.edu/courses/6-006-introduction-to-algorithms-spring-2020/)
- [동빈나 — 자료구조 기초](https://www.youtube.com/@dongbinna)

### 시각화 / 실습
- [VisuAlgo — Sorting](https://visualgo.net/en/sorting) (배열 정렬 시각)
- [Python list 내부 구현 (CPython source)](https://github.com/python/cpython/blob/main/Objects/listobject.c)
- [Java ArrayList source](https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/util/ArrayList.java)

### 한국어
- "Do it! 자료구조와 함께 배우는 알고리즘 입문" (한빛미디어)
- 인프런 자료구조 강의 다수

---

## 15. 관련

- [[../linked-lists/linked-lists]] — 동적 크기 + O(1) 삽입 (다음 노트)
- [[../hash-tables/hash-tables]] — O(1) 키 매핑
- [[../../algorithm/sorting/sorting]] — 배열 정렬
- [[../../algorithm/binary-search/binary-search]] — 정렬된 배열 검색
- [[../data-structure|↑ data-structure 인덱스]]
