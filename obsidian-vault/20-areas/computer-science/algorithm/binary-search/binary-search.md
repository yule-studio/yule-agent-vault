---
title: "이진 탐색 (Binary Search)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T02:00:00+09:00
tags:
  - algorithm
  - binary-search
---

# 이진 탐색 (Binary Search)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 수준 + 11 언어 (이코테 Ch.07/15) |

**[[../algorithm|↑ algorithm]]** · **[[../computer-science/computer-science|↑↑ computer-science]]**

---

## 1. 한 줄 정의

**정렬된 데이터** 에서 **범위를 절반씩 좁히며** 원하는 값을 `O(log N)` 에 찾는다. **파라메트릭 서치 (Parametric Search)** — "답을 직접 이분 탐색" — 는
더 강력한 응용.

---

## 2. 이진 탐색은 언제 쓰는가? — 시그널 인식

### (a) 직접 이분 탐색

- **정렬된 배열에서 값 찾기** — `O(log N)`
- **삽입 위치 찾기** — `bisect_left / right`
- **중복 값의 첫 / 마지막 위치**
- **lower_bound / upper_bound**

### (b) 파라메트릭 서치 (답 자체를 이분 탐색)

가장 강력한 응용. "X 가 가능한가?" 가 단조 (monotone) 일 때:

- **"최대 X 의 최솟값" / "최소 X 의 최댓값"** — 단조함이 보이면 100% 파라메트릭.
- **떡볶이 떡 길이 / 공유기 거리 / 나무 자르기** 같은 문제.

### (c) 신호 키워드

- "N 개 중 ... 찾아라" + N ≥ 10^5 → O(log N) 필요
- "최대의 최솟값 / 최소의 최댓값" — Parametric
- 입력 크기 N = 10^9 — 정렬·완탐 불가, **이분 탐색만**
- "정확히 X 가 몇 개?" — `upper_bound - lower_bound`

### (d) 다른 알고리즘과 구분

| 차원 | 이분 탐색 | 투 포인터 | 슬라이딩 윈도우 |
| --- | --- | --- | --- |
| 시간 | O(log N) | O(N) | O(N) |
| 전제 | 정렬 필요 | 정렬 또는 단조성 | 연속 부분 |
| 신호 | 큰 N + 정렬 | 합/곱 + 정렬 | 연속 구간 길이 |

---

## 3. 핵심 직관

**전화번호부 찾기**:
- 책 가운데를 펴서 → 내 이름이 앞이면 앞쪽 절반, 뒤면 뒤쪽 절반.
- 한 번에 후보가 반으로 줄어 → log₂(N) 번이면 끝.

**N=10^9 라도 30 번만에 답** (2^30 ≈ 10^9).

**무거운 직관**: 단조 함수에서 "true → false 가 바뀌는 지점" 찾기.

---

## 4. 동작 원리 — 예시

### (a) 표준 이분 탐색 — 7 찾기

```
정렬 배열: [1, 3, 5, 7, 9, 11, 13, 15], 타겟 = 7

Step 1: lo=0, hi=7, mid=3, arr[3]=7 == target → 찾음! (idx 3)
```

### (b) Lower bound — "7 이상의 첫 위치"

```
배열: [1, 3, 5, 7, 7, 7, 11, 15], 타겟 = 7

Step 1: lo=0, hi=8, mid=4, arr[4]=7 >= 7 → hi=4
Step 2: lo=0, hi=4, mid=2, arr[2]=5 < 7 → lo=3
Step 3: lo=3, hi=4, mid=3, arr[3]=7 >= 7 → hi=3
Step 4: lo=3, hi=3 → 종료. 답=3
```

### (c) Upper bound — "7 보다 큰 첫 위치"

```
배열: [1, 3, 5, 7, 7, 7, 11, 15], 타겟 = 7

Step 1: lo=0, hi=8, mid=4, arr[4]=7 <= 7 → lo=5
Step 2: lo=5, hi=8, mid=6, arr[6]=11 > 7 → hi=6
Step 3: lo=5, hi=6, mid=5, arr[5]=7 <= 7 → lo=6
Step 4: lo=6, hi=6 → 종료. 답=6
```

**Count of 7** = upper_bound - lower_bound = 6 - 3 = 3.

### (d) 파라메트릭 서치 — 떡볶이 떡 (Ch.07)

떡 길이 `[19, 15, 10, 17]`, 손님이 6 cm 가져가게 자를 때 H 의 최댓값?

```
가능 여부 (H) = 자른 떡 합 >= 6:
- H = 19: 합 = 0+0+0+0 = 0 < 6 → False
- H = 18: 합 = 1+0+0+0 = 1 < 6 → False
- H = 15: 합 = 4+0+0+2 = 6 >= 6 → True
- H = 14: 합 = 5+1+0+3 = 9 >= 6 → True

→ "True 가 되는 가장 큰 H" 를 찾자 = 15
```

```
이분 탐색 (H 의 범위 0..19):
Step 1: lo=0, hi=19, mid=9. cut(9)=10+6+1+8 >= 6 → 답=9, lo=10
Step 2: lo=10, hi=19, mid=14. cut(14)=5+1+0+3 >= 6 → 답=14, lo=15
Step 3: lo=15, hi=19, mid=17. cut(17)=2+0+0+0 < 6 → hi=16
Step 4: lo=15, hi=16, mid=15. cut(15)=4+0+0+2 >= 6 → 답=15, lo=16
Step 5: lo=16, hi=16 → 종료. 답=15
```

---

## 5. 복잡도

| 변형 | 시간 | 공간 |
| --- | --- | --- |
| 정렬된 배열 값 탐색 | `O(log N)` | `O(1)` |
| 정렬 + 이분 탐색 | `O(N log N)` | `O(1)` |
| 파라메트릭 (답 범위 [lo, hi]) | `O(log(hi-lo) × check)` | check 함수에 의존 |
| 이진 탐색 트리 (BST) | `O(log N)` 평균, `O(N)` 최악 | `O(N)` |

---

## 6. 의사 코드

### 표준 이분 탐색
```
function BSEARCH(arr, target):
    lo ← 0
    hi ← len(arr) - 1
    while lo <= hi:
        mid ← (lo + hi) / 2
        if arr[mid] == target: return mid
        elif arr[mid] < target: lo ← mid + 1
        else: hi ← mid - 1
    return -1
```

### Lower bound
```
function LOWER_BOUND(arr, target):
    lo ← 0, hi ← len(arr)
    while lo < hi:
        mid ← (lo + hi) / 2
        if arr[mid] >= target: hi ← mid
        else: lo ← mid + 1
    return lo   # target 이상의 첫 위치 (없으면 len)
```

### 파라메트릭 서치
```
function PARAMETRIC(lo, hi, condition):
    result ← lo - 1
    while lo <= hi:
        mid ← (lo + hi) / 2
        if condition(mid):  # True 영역
            result ← mid
            lo ← mid + 1     # 더 큰 답 찾기 (또는 hi ← mid - 1)
        else:
            hi ← mid - 1
    return result
```

---

## 7. 언어별 구현 — 표준 이분 탐색 + Lower Bound + 파라메트릭 (11 언어)

### Python 3
```python
import bisect

# 표준
def binary_search(arr, target):
    lo, hi = 0, len(arr) - 1
    while lo <= hi:
        mid = (lo + hi) // 2
        if arr[mid] == target: return mid
        elif arr[mid] < target: lo = mid + 1
        else: hi = mid - 1
    return -1

# Lower / Upper bound — 내장 사용
arr = [1, 3, 5, 7, 7, 7, 11, 15]
print(bisect.bisect_left(arr, 7))    # 3
print(bisect.bisect_right(arr, 7))   # 6

# 파라메트릭 — 떡볶이 떡
def can_cut(heights, h, target):
    return sum(max(0, x - h) for x in heights) >= target

def max_h(heights, target):
    lo, hi = 0, max(heights)
    result = 0
    while lo <= hi:
        mid = (lo + hi) // 2
        if can_cut(heights, mid, target):
            result = mid
            lo = mid + 1
        else:
            hi = mid - 1
    return result

print(max_h([19, 15, 10, 17], 6))   # 15
```

### Java
```java
import java.util.*;
public class BSearch {
    static int bsearch(int[] a, int target) {
        int lo = 0, hi = a.length - 1;
        while (lo <= hi) {
            int mid = lo + (hi - lo) / 2;
            if (a[mid] == target) return mid;
            else if (a[mid] < target) lo = mid + 1;
            else hi = mid - 1;
        }
        return -1;
    }
    static int lowerBound(int[] a, int target) {
        int lo = 0, hi = a.length;
        while (lo < hi) {
            int mid = lo + (hi - lo) / 2;
            if (a[mid] >= target) hi = mid;
            else lo = mid + 1;
        }
        return lo;
    }

    static int maxH(int[] h, int target) {
        int lo = 0, hi = Arrays.stream(h).max().getAsInt(), result = 0;
        while (lo <= hi) {
            int mid = lo + (hi - lo) / 2;
            long sum = 0;
            for (int x : h) sum += Math.max(0, x - mid);
            if (sum >= target) { result = mid; lo = mid + 1; }
            else hi = mid - 1;
        }
        return result;
    }

    public static void main(String[] args) {
        int[] arr = {1,3,5,7,7,7,11,15};
        System.out.println(bsearch(arr, 7));               // any of 3,4,5
        System.out.println(lowerBound(arr, 7));            // 3
        // Arrays.binarySearch 도 가능 (찾으면 idx, 없으면 -(삽입위치+1))
        System.out.println(Arrays.binarySearch(arr, 7));   // 5 (any)
        System.out.println(maxH(new int[]{19,15,10,17}, 6));  // 15
    }
}
```

### C++
```cpp
#include <bits/stdc++.h>
using namespace std;

int bsearch(vector<int>& a, int target) {
    int lo = 0, hi = a.size() - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (a[mid] == target) return mid;
        else if (a[mid] < target) lo = mid + 1;
        else hi = mid - 1;
    }
    return -1;
}

int main() {
    vector<int> arr = {1,3,5,7,7,7,11,15};
    // STL 함수 — 정렬된 배열 가정
    auto lb = lower_bound(arr.begin(), arr.end(), 7);   // 3 위치 iterator
    auto ub = upper_bound(arr.begin(), arr.end(), 7);   // 6 위치
    cout << (lb - arr.begin()) << " " << (ub - arr.begin()) << endl;
    cout << bsearch(arr, 7) << endl;

    // 파라메트릭 - 떡볶이 떡
    vector<int> h = {19, 15, 10, 17};
    int target = 6;
    int lo = 0, hi = *max_element(h.begin(), h.end()), result = 0;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        long long sum = 0;
        for (int x : h) sum += max(0, x - mid);
        if (sum >= target) { result = mid; lo = mid + 1; }
        else hi = mid - 1;
    }
    cout << result << endl;
}
```

### C
```c
#include <stdio.h>
#include <stdlib.h>

int bsearch_iter(int *a, int n, int target) {
    int lo = 0, hi = n - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (a[mid] == target) return mid;
        else if (a[mid] < target) lo = mid + 1;
        else hi = mid - 1;
    }
    return -1;
}

int lower_bound(int *a, int n, int target) {
    int lo = 0, hi = n;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (a[mid] >= target) hi = mid;
        else lo = mid + 1;
    }
    return lo;
}

int max_h(int *h, int n, int target) {
    int lo = 0, hi = 0;
    for (int i = 0; i < n; i++) if (h[i] > hi) hi = h[i];
    int result = 0;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        long long sum = 0;
        for (int i = 0; i < n; i++) {
            int diff = h[i] - mid;
            if (diff > 0) sum += diff;
        }
        if (sum >= target) { result = mid; lo = mid + 1; }
        else hi = mid - 1;
    }
    return result;
}

int main() {
    int arr[] = {1,3,5,7,7,7,11,15};
    printf("%d\n", lower_bound(arr, 8, 7));     // 3
    int h[] = {19, 15, 10, 17};
    printf("%d\n", max_h(h, 4, 6));              // 15
}
```

### JavaScript (Node.js)
```javascript
function binarySearch(arr, target) {
    let lo = 0, hi = arr.length - 1;
    while (lo <= hi) {
        const mid = Math.floor((lo + hi) / 2);
        if (arr[mid] === target) return mid;
        else if (arr[mid] < target) lo = mid + 1;
        else hi = mid - 1;
    }
    return -1;
}

function lowerBound(arr, target) {
    let lo = 0, hi = arr.length;
    while (lo < hi) {
        const mid = Math.floor((lo + hi) / 2);
        if (arr[mid] >= target) hi = mid;
        else lo = mid + 1;
    }
    return lo;
}

// 파라메트릭
function maxH(h, target) {
    let lo = 0, hi = Math.max(...h), result = 0;
    while (lo <= hi) {
        const mid = Math.floor((lo + hi) / 2);
        const sum = h.reduce((s, x) => s + Math.max(0, x - mid), 0);
        if (sum >= target) { result = mid; lo = mid + 1; }
        else hi = mid - 1;
    }
    return result;
}

console.log(lowerBound([1,3,5,7,7,7,11,15], 7));    // 3
console.log(maxH([19,15,10,17], 6));                 // 15
```

### TypeScript
```typescript
function binarySearch(arr: number[], target: number): number {
    let lo = 0, hi = arr.length - 1;
    while (lo <= hi) {
        const mid = Math.floor((lo + hi) / 2);
        if (arr[mid] === target) return mid;
        else if (arr[mid] < target) lo = mid + 1;
        else hi = mid - 1;
    }
    return -1;
}

function lowerBound(arr: number[], target: number): number {
    let lo = 0, hi = arr.length;
    while (lo < hi) {
        const mid = Math.floor((lo + hi) / 2);
        if (arr[mid] >= target) hi = mid;
        else lo = mid + 1;
    }
    return lo;
}
```

### Kotlin
```kotlin
fun binarySearch(a: IntArray, target: Int): Int {
    var lo = 0; var hi = a.size - 1
    while (lo <= hi) {
        val mid = lo + (hi - lo) / 2
        when {
            a[mid] == target -> return mid
            a[mid] < target -> lo = mid + 1
            else -> hi = mid - 1
        }
    }
    return -1
}

// 내장 (정렬된 배열)
// Arrays.binarySearch — 음수면 -(insertion point + 1)
fun main() {
    val arr = intArrayOf(1,3,5,7,7,7,11,15)
    println(binarySearch(arr, 7))
    println(arr.toList().binarySearch(7))   // Kotlin std

    // 파라메트릭
    val h = intArrayOf(19, 15, 10, 17)
    var lo = 0; var hi = h.max()!!; var result = 0
    while (lo <= hi) {
        val mid = lo + (hi - lo) / 2
        val sum = h.sumOf { maxOf(0, it - mid).toLong() }
        if (sum >= 6) { result = mid; lo = mid + 1 } else hi = mid - 1
    }
    println(result)   // 15
}
```

### Swift
```swift
func binarySearch(_ arr: [Int], _ target: Int) -> Int {
    var lo = 0, hi = arr.count - 1
    while lo <= hi {
        let mid = lo + (hi - lo) / 2
        if arr[mid] == target { return mid }
        else if arr[mid] < target { lo = mid + 1 }
        else { hi = mid - 1 }
    }
    return -1
}

func lowerBound(_ arr: [Int], _ target: Int) -> Int {
    var lo = 0, hi = arr.count
    while lo < hi {
        let mid = lo + (hi - lo) / 2
        if arr[mid] >= target { hi = mid } else { lo = mid + 1 }
    }
    return lo
}

let h = [19, 15, 10, 17]
var lo = 0, hi = h.max()!, result = 0
while lo <= hi {
    let mid = lo + (hi - lo) / 2
    let sum = h.reduce(0) { $0 + max(0, $1 - mid) }
    if sum >= 6 { result = mid; lo = mid + 1 } else { hi = mid - 1 }
}
print(result)
```

### Go
```go
package main

import (
    "fmt"
    "sort"
)

func binarySearch(a []int, target int) int {
    lo, hi := 0, len(a)-1
    for lo <= hi {
        mid := lo + (hi-lo)/2
        if a[mid] == target { return mid }
        if a[mid] < target { lo = mid + 1 } else { hi = mid - 1 }
    }
    return -1
}

func main() {
    arr := []int{1,3,5,7,7,7,11,15}
    // sort.SearchInts == lower_bound
    fmt.Println(sort.SearchInts(arr, 7))   // 3

    h := []int{19, 15, 10, 17}
    target := 6
    maxH := 0
    for _, x := range h { if x > maxH { maxH = x } }
    lo, hi, result := 0, maxH, 0
    for lo <= hi {
        mid := lo + (hi-lo)/2
        sum := 0
        for _, x := range h { if x > mid { sum += x - mid } }
        if sum >= target { result = mid; lo = mid + 1 } else { hi = mid - 1 }
    }
    fmt.Println(result)
}
```

### Rust
```rust
fn binary_search(a: &[i32], target: i32) -> Option<usize> {
    let (mut lo, mut hi) = (0, a.len());
    while lo < hi {
        let mid = lo + (hi - lo) / 2;
        if a[mid] == target { return Some(mid); }
        else if a[mid] < target { lo = mid + 1; }
        else { hi = mid; }
    }
    None
}

fn max_h(h: &[i32], target: i64) -> i32 {
    let (mut lo, mut hi) = (0, *h.iter().max().unwrap());
    let mut result = 0;
    while lo <= hi {
        let mid = lo + (hi - lo) / 2;
        let sum: i64 = h.iter().map(|&x| (x - mid).max(0) as i64).sum();
        if sum >= target { result = mid; lo = mid + 1; }
        else { hi = mid - 1; }
    }
    result
}

fn main() {
    let arr = [1,3,5,7,7,7,11,15];
    println!("{:?}", arr.binary_search(&7));   // 내장
    println!("{}", max_h(&[19, 15, 10, 17], 6));
}
```

### C#
```csharp
using System;

public class BSearch {
    public static int BinarySearch(int[] a, int target) {
        int lo = 0, hi = a.Length - 1;
        while (lo <= hi) {
            int mid = lo + (hi - lo) / 2;
            if (a[mid] == target) return mid;
            else if (a[mid] < target) lo = mid + 1;
            else hi = mid - 1;
        }
        return -1;
    }

    public static int MaxH(int[] h, int target) {
        int lo = 0, hi = int.MinValue, result = 0;
        foreach (var x in h) if (x > hi) hi = x;
        while (lo <= hi) {
            int mid = lo + (hi - lo) / 2;
            long sum = 0;
            foreach (var x in h) if (x > mid) sum += x - mid;
            if (sum >= target) { result = mid; lo = mid + 1; }
            else hi = mid - 1;
        }
        return result;
    }

    public static void Main() {
        int[] arr = {1,3,5,7,7,7,11,15};
        // 내장
        Console.WriteLine(Array.BinarySearch(arr, 7));  // 음수면 ~insertion
        Console.WriteLine(MaxH(new[] {19,15,10,17}, 6));
    }
}
```

---

## 8. 변형 / 응용

### (a) 회전된 정렬 배열에서 탐색
`[7,8,9,1,2,3,4]` 같은 회전 배열. 어느 쪽이 정렬됐는지 판단 + 이분.

### (b) Peak Finding
인접보다 큰 원소 찾기. `O(log N)`.

### (c) 제곱근 / 거듭제곱 (이분)
`sqrt(N)` 을 이분으로 — `lo*lo <= N` 만족하는 최댓값.

### (d) 트리에서 K 번째 원소
in-order 진행 카운트.

### (e) 분수 / 실수 이분 탐색
`hi - lo > 1e-9` 까지 반복. 100 회 정도면 충분.

### (f) 매개 변수 탐색 (Parametric Search) 패턴
- "X 의 최댓값" + 단조 조건 → 이분.
- "X 의 최솟값" + 단조 조건 → 이분.
- check 함수가 단조 (monotone) 임이 핵심.

---

## 9. 함정 / 안티패턴

### 함정 1 — `(lo + hi) / 2` 오버플로

Java / C / C++ 의 `int` 에서 `lo + hi` 가 INT_MAX 초과 → 음수. 안전한 방식:

```cpp
int mid = lo + (hi - lo) / 2;   // 오버플로 방지
```

### 함정 2 — 종료 조건 / 인덱스 범위

```python
# 닫힌 범위 [lo, hi]
lo, hi = 0, n - 1
while lo <= hi: ...

# 반열린 범위 [lo, hi)
lo, hi = 0, n
while lo < hi: ...
```

**둘 중 하나 일관**. 섞으면 OB 또는 무한 루프.

### 함정 3 — 정렬 안 한 배열에 이분

이분 탐색은 **정렬 전제**. 반드시 `sort()` 먼저.

### 함정 4 — 무한 루프

`lo = mid` 또는 `hi = mid` 같이 범위가 안 줄어들면 무한 루프.

```python
# 무한 루프
while lo < hi:
    mid = (lo + hi) // 2
    if cond: lo = mid       # lo 가 mid 와 같으면 안 줄어
    else: hi = mid

# 옳음 — mid + 1 또는 mid - 1
while lo < hi:
    mid = (lo + hi) // 2
    if cond: lo = mid + 1
    else: hi = mid
```

### 함정 5 — Lower/Upper bound 헷갈림

- `lower_bound(target)` = "target 이상의 첫 위치"
- `upper_bound(target)` = "target 보다 큰 첫 위치"
- **개수**: `upper - lower`
- **존재 여부**: `lower 가 끝이 아니고 arr[lower] == target`

### 함정 6 — 파라메트릭에서 답의 범위

```python
# 잘못
lo, hi = min(h), max(h)
# H = 0 도 답이 될 수 있는데 lo = min(h) 면 못 본다.

# 옳음
lo, hi = 0, max(h)
```

### 함정 7 — `bisect.bisect_left` 의 key 매개변수

Python 3.10+ 부터 `key` 지원. 그 이전엔 별도 추출 배열 필요.

### 함정 8 — JavaScript `Math.floor` 빠뜨림

```javascript
const mid = (lo + hi) / 2;   // 실수
arr[mid]   // 비정상

const mid = Math.floor((lo + hi) / 2);   // 옳음
// 또는: const mid = (lo + hi) >> 1;
```

---

## 10. 실전 풀이 절차 — 6 단계

1. **이분 탐색이 필요한가?** — N ≥ 10^5 + 정렬 가능 / 단조 신호.
2. **표준 vs 파라메트릭** — 값을 찾나, 답을 찾나.
3. **범위 설정** — `[lo, hi]` 의 정확한 경계.
4. **check 함수** — `cond(mid)` 가 단조 함수인지 검증.
5. **종료 조건** — `lo <= hi` or `lo < hi` 일관.
6. **답 추출** — `lo` / `hi` / `result` 중 어느 것을 반환할지.

---

## 11. 대표 문제 (이코테 Ch.07 + Ch.15)

### Q1. 부품 찾기 (Ch.07)
N 개 부품 vs M 개 요청. 각 요청이 부품에 있는지.

```python
def solution(n, parts, m, requests):
    parts.sort()
    result = []
    for req in requests:
        # 이분 탐색
        lo, hi = 0, n - 1
        found = False
        while lo <= hi:
            mid = (lo + hi) // 2
            if parts[mid] == req:
                found = True; break
            elif parts[mid] < req: lo = mid + 1
            else: hi = mid - 1
        result.append('yes' if found else 'no')
    return result
```
**복잡도**: O((N+M) log N).

### Q2. 떡볶이 떡 만들기 (Ch.07) — 파라메트릭
위 예시 코드 참조.

### Q3. 정렬된 배열에서 특정 수의 개수 (Ch.15)

```python
from bisect import bisect_left, bisect_right
n, x = map(int, input().split())
arr = list(map(int, input().split()))
count = bisect_right(arr, x) - bisect_left(arr, x)
print(count if count > 0 else -1)
```

### Q4. 고정점 찾기 (Ch.15)
arr[i] == i 인 인덱스.

```python
def find_fixed(arr):
    lo, hi = 0, len(arr) - 1
    while lo <= hi:
        mid = (lo + hi) // 2
        if arr[mid] == mid: return mid
        elif arr[mid] < mid: lo = mid + 1
        else: hi = mid - 1
    return -1
```

### Q5. 공유기 설치 (Ch.15) — 파라메트릭

집 N 개에 공유기 C 개를 설치. 인접 공유기 거리 최댓값.

```python
def solution(n, c, houses):
    houses.sort()
    lo, hi = 1, houses[-1] - houses[0]
    result = 0
    while lo <= hi:
        mid = (lo + hi) // 2
        count = 1
        prev = houses[0]
        for h in houses[1:]:
            if h - prev >= mid:
                count += 1
                prev = h
        if count >= c:
            result = mid
            lo = mid + 1
        else:
            hi = mid - 1
    return result
```

### Q6. 가사 검색 (Ch.15, 카카오) — 트라이 / 이분

`fro??` 같은 와일드카드 단어 찾기. 단어 길이별로 그룹 + 정렬 + 이분 (또는 트라이).

---

## 12. 연습 문제 (난이도별)

### 입문
- [백준 1920 수 찾기](https://www.acmicpc.net/problem/1920)
- [백준 10816 숫자 카드 2](https://www.acmicpc.net/problem/10816) — count
- [백준 1654 랜선 자르기](https://www.acmicpc.net/problem/1654) — 파라메트릭

### 중급
- [백준 2805 나무 자르기](https://www.acmicpc.net/problem/2805) — 떡볶이 떡 유사
- [백준 2110 공유기 설치](https://www.acmicpc.net/problem/2110)
- [백준 1300 K 번째 수](https://www.acmicpc.net/problem/1300)

### 고급
- [백준 12015 가장 긴 증가하는 부분 수열 2](https://www.acmicpc.net/problem/12015) — LIS + 이분
- [백준 1939 중량 제한](https://www.acmicpc.net/problem/1939) — 이분 + BFS
- [백준 3079 입국 심사](https://www.acmicpc.net/problem/3079) — 파라메트릭

---

## 13. 학습 자료

- 이코테 **Ch.07, 15**
- [동빈나 — 이진 탐색 영상](https://www.youtube.com/watch?v=94RC-DsGMLo)
- [CLRS 학술](https://mitpress.mit.edu/9780262046305/) — 분할 정복 Chapter
- [VisuAlgo — Binary Search](https://visualgo.net/en/bst)
- [Powerful Ultimate Binary Search Template](https://leetcode.com/discuss/general-discussion/786126/) — LeetCode

---

## 14. 관련

- [[../sorting/sorting]] — 정렬이 이분 탐색의 전제
- [[../dynamic-programming/dynamic-programming]] — LIS 등 이분 + DP
- [[../algorithm|↑ algorithm 인덱스]]
