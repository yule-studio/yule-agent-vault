---
title: "정렬 (Sorting)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T01:30:00+09:00
tags:
  - algorithm
  - sorting
---

# 정렬 (Sorting)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 수준 + 11 언어 (이코테 Ch.06/14) |

**[[../algorithm|↑ algorithm]]** · **[[../computer-science/computer-science|↑↑ computer-science]]**

---

## 1. 한 줄 정의

**N 개의 데이터를 특정 기준 (오름차순 / 내림차순)** 으로 나열하는 알고리즘.
거의 모든 알고리즘 문제의 **전처리** 또는 **핵심 단계**.

---

## 2. 정렬은 언제 쓰는가? — 시그널 인식

### (a) 정렬이 등장하는 패턴

- **이분 탐색 전처리** — 정렬되지 않으면 이분 탐색 불가
- **그리디 알고리즘** — "끝나는 시간 빠른 순" 등
- **중복 제거 / 유일값** — 정렬 후 인접 비교
- **위/아래 K 개** — 정렬 후 슬라이싱
- **순위 매기기** — 정렬 후 인덱스
- **lazy interval problem** — 시작/끝 정렬 후 sweep

### (b) 어떤 정렬을 쓸 것인가?

| 정렬 | 시간 (평균/최악) | 공간 | 안정성 | 언제 |
| --- | --- | --- | --- | --- |
| 선택 정렬 | O(N²) | O(1) | ❌ | 학습용. 실무 X |
| 삽입 정렬 | O(N²)/O(N) | O(1) | ✅ | 거의 정렬된 데이터 |
| 버블 정렬 | O(N²) | O(1) | ✅ | 학습용. 실무 X |
| 퀵 정렬 | O(N log N)/O(N²) | O(log N) | ❌ | 평균 가장 빠름. 표준 |
| 병합 정렬 | O(N log N)/O(N log N) | O(N) | ✅ | 안정 + 최악 보장 |
| 힙 정렬 | O(N log N)/O(N log N) | O(1) | ❌ | 우선순위 큐 |
| 계수 정렬 | O(N+K) | O(N+K) | ✅ | 정수 범위 작을 때 |
| 기수 정렬 | O(d×(N+K)) | O(N+K) | ✅ | 정수 자릿수 적을 때 |
| Timsort | O(N log N) | O(N) | ✅ | **Python `sort()` 기본** |

**실전 99%**: 언어 내장 정렬 (`sort()` / `Arrays.sort()` / `std::sort()`).

### (c) 코딩 테스트 — 거의 항상 내장 정렬

직접 구현 ≠ 시험 목적. 내장이 빠르고 안정. 단, **정렬 원리 + 안정성 + 커스텀
비교 함수** 를 면접에서 물을 수 있음.

---

## 3. 핵심 직관 — 5 가지 기본 정렬

### 선택 정렬 (Selection Sort)
"매번 가장 작은 걸 골라 앞에 놓는다."
- 매 회전: 미정렬 중 최솟값 찾기 → 맨 앞과 swap.
- O(N²) 비교, O(N) swap.

### 삽입 정렬 (Insertion Sort)
"카드를 한 장씩 들어 정렬된 손에 끼워 넣는다."
- 왼쪽이 항상 정렬됨. 새 카드를 적절한 위치에 삽입.
- **거의 정렬된 데이터** 에 O(N) 으로 빠름.

### 퀵 정렬 (Quick Sort)
"기준값 (pivot) 보다 작은 건 왼쪽, 큰 건 오른쪽."
- 분할 정복 (divide & conquer).
- 평균 O(N log N), 최악 O(N²) — pivot 선택이 잘못되면.
- 실무 표준 (개선판 사용).

### 병합 정렬 (Merge Sort)
"반으로 쪼개서 각자 정렬한 뒤 합친다."
- 분할 정복 + 안정 정렬.
- 항상 O(N log N) — 최악 보장.
- 추가 메모리 O(N) 필요.

### 계수 정렬 (Counting Sort)
"값 범위가 작을 때 카운트만 세서 정렬."
- 값이 0..K 사이면 O(N+K).
- K 가 크면 메모리 폭발 — N=100, K=10^9 불가.

---

## 4. 동작 원리 — 예시 [7, 5, 9, 0, 3, 1, 6, 2, 4, 8]

### 선택 정렬 (10 단계 중 첫 3 회전)
```
초기:  [7, 5, 9, 0, 3, 1, 6, 2, 4, 8]
1회전: min=0 (idx 3) ↔ [0]. → [0, 5, 9, 7, 3, 1, 6, 2, 4, 8]
2회전: min=1 (idx 5) ↔ [1]. → [0, 1, 9, 7, 3, 5, 6, 2, 4, 8]
3회전: min=2 (idx 7) ↔ [2]. → [0, 1, 2, 7, 3, 5, 6, 9, 4, 8]
...
최종:  [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
```

### 삽입 정렬
```
초기:    [7, 5, 9, 0, 3, 1, 6, 2, 4, 8]
i=1, 5:  5 < 7 → 앞으로. [5, 7, 9, 0, 3, ...]
i=2, 9:  9 > 7 → 제자리.  [5, 7, 9, 0, 3, ...]
i=3, 0:  0 < 5 → 맨 앞으로. [0, 5, 7, 9, 3, ...]
i=4, 3:  3 ↔ 5. [0, 3, 5, 7, 9, 1, ...]
...
```

### 퀵 정렬 (Hoare 파티션 변형 — pivot=첫 원소)
```
초기:  [7, 5, 9, 0, 3, 1, 6, 2, 4, 8], pivot=7
       L: 9 (7 보다 큰 첫)
       R: 4 (7 보다 작은 마지막)
       swap(9, 4) → [7, 5, 4, 0, 3, 1, 6, 2, 9, 8]
       L: 5 → idx 1, R: 2 → idx 7. ...
       엇갈리면 pivot ↔ R: [2, 5, 4, 0, 3, 1, 6, 7, 9, 8]
       이제 [2,5,4,0,3,1,6] | [7] | [9,8] 로 나뉨
       재귀 정렬.
```

### 병합 정렬
```
[7, 5, 9, 0, 3, 1, 6, 2, 4, 8]
   ↓ 분할
[7, 5, 9, 0, 3] [1, 6, 2, 4, 8]
   ↓ 더 분할
[7,5] [9,0,3]  [1,6] [2,4,8]
   ↓ ...
[5,7] [0,3,9]  [1,6] [2,4,8]
   ↓ 병합
[0,3,5,7,9]   [1,2,4,6,8]
   ↓ 최종 병합
[0,1,2,3,4,5,6,7,8,9]
```

### 계수 정렬 (값 범위 0..9)
```
초기 배열: [7,5,9,0,3,1,6,2,4,8]
count =    [1,1,1,1,1,1,1,1,1,1]
            0 1 2 3 4 5 6 7 8 9
출력: 각 값을 count 번 출력 → [0,1,2,3,4,5,6,7,8,9]
```

---

## 5. 안정성 (Stability)

**같은 값이 입력 순서대로 유지**되면 안정 정렬.

```
입력: [(A,3), (B,1), (C,3), (D,2)]
안정:  [(B,1), (D,2), (A,3), (C,3)]   ← A 가 C 보다 먼저
불안정: [(B,1), (D,2), (C,3), (A,3)]   ← 순서 바뀜 가능
```

**왜 중요?** 다중 기준 정렬 — 먼저 부수 기준 정렬 후 주 기준으로 안정 정렬
하면 부수 기준이 보존됨. 예: 학생을 점수 + 이름 순으로 정렬할 때.

---

## 6. 의사 코드

### 선택 정렬
```
for i in 0..N-1:
    min_idx ← i
    for j in i+1..N:
        if a[j] < a[min_idx]: min_idx ← j
    swap(a[i], a[min_idx])
```

### 삽입 정렬
```
for i in 1..N:
    key ← a[i]
    j ← i - 1
    while j >= 0 and a[j] > key:
        a[j+1] ← a[j]
        j ← j - 1
    a[j+1] ← key
```

### 퀵 정렬
```
function QSORT(a, lo, hi):
    if lo < hi:
        p ← partition(a, lo, hi)
        QSORT(a, lo, p-1)
        QSORT(a, p+1, hi)

function partition(a, lo, hi):   # Lomuto
    pivot ← a[hi]
    i ← lo - 1
    for j in lo..hi-1:
        if a[j] <= pivot:
            i ← i + 1
            swap(a[i], a[j])
    swap(a[i+1], a[hi])
    return i + 1
```

### 병합 정렬
```
function MSORT(a, lo, hi):
    if lo < hi:
        mid ← (lo + hi) / 2
        MSORT(a, lo, mid)
        MSORT(a, mid+1, hi)
        merge(a, lo, mid, hi)
```

---

## 7. 언어별 구현 — 퀵 정렬 + 내장 정렬 (11 언어)

### Python 3
```python
# 내장 (Timsort, 안정, O(N log N))
data = [7, 5, 9, 0, 3, 1, 6, 2, 4, 8]
sorted_asc = sorted(data)
sorted_desc = sorted(data, reverse=True)
data.sort(key=lambda x: -x)   # in-place + 키

# 커스텀 비교 (학생 점수 내림 + 이름 오름)
students = [('Alice', 90), ('Bob', 80), ('Cathy', 90)]
students.sort(key=lambda s: (-s[1], s[0]))   # [(Alice,90), (Cathy,90), (Bob,80)]

# 퀵 정렬 직접
def quick_sort(arr):
    if len(arr) <= 1: return arr
    pivot = arr[0]
    tail = arr[1:]
    left = [x for x in tail if x <= pivot]
    right = [x for x in tail if x > pivot]
    return quick_sort(left) + [pivot] + quick_sort(right)

print(quick_sort([7,5,9,0,3,1,6,2,4,8]))
```

### Java
```java
import java.util.*;
public class SortDemo {
    public static void main(String[] args) {
        int[] data = {7,5,9,0,3,1,6,2,4,8};
        Arrays.sort(data);
        System.out.println(Arrays.toString(data));

        Integer[] dataInt = {7,5,9,0,3,1,6,2,4,8};
        Arrays.sort(dataInt, Collections.reverseOrder());

        // 커스텀 정렬
        int[][] students = {{90,1}, {80,2}, {90,3}};
        Arrays.sort(students, (a, b) -> a[0] != b[0] ? b[0] - a[0] : a[1] - b[1]);
    }

    static void quickSort(int[] a, int lo, int hi) {
        if (lo < hi) {
            int p = partition(a, lo, hi);
            quickSort(a, lo, p - 1);
            quickSort(a, p + 1, hi);
        }
    }
    static int partition(int[] a, int lo, int hi) {
        int pivot = a[hi], i = lo - 1;
        for (int j = lo; j < hi; j++)
            if (a[j] <= pivot) { i++; int t = a[i]; a[i] = a[j]; a[j] = t; }
        int t = a[i+1]; a[i+1] = a[hi]; a[hi] = t;
        return i + 1;
    }
}
```

### C++
```cpp
#include <bits/stdc++.h>
using namespace std;

void quick_sort(vector<int>& a, int lo, int hi) {
    if (lo < hi) {
        int pivot = a[hi], i = lo - 1;
        for (int j = lo; j < hi; j++)
            if (a[j] <= pivot) swap(a[++i], a[j]);
        swap(a[i+1], a[hi]);
        quick_sort(a, lo, i);
        quick_sort(a, i+2, hi);
    }
}

int main() {
    vector<int> data = {7,5,9,0,3,1,6,2,4,8};
    sort(data.begin(), data.end());                  // 오름차순 (introsort)
    sort(data.begin(), data.end(), greater<int>());  // 내림차순

    vector<pair<int,string>> students = {{90,"Alice"},{80,"Bob"},{90,"Cathy"}};
    sort(students.begin(), students.end(), [](auto& a, auto& b){
        if (a.first != b.first) return a.first > b.first;
        return a.second < b.second;
    });
}
```

### C
```c
#include <stdio.h>
#include <stdlib.h>

void quick_sort(int *a, int lo, int hi) {
    if (lo < hi) {
        int pivot = a[hi], i = lo - 1, t;
        for (int j = lo; j < hi; j++)
            if (a[j] <= pivot) { i++; t=a[i]; a[i]=a[j]; a[j]=t; }
        t = a[i+1]; a[i+1] = a[hi]; a[hi] = t;
        quick_sort(a, lo, i);
        quick_sort(a, i+2, hi);
    }
}

int cmp_asc(const void *a, const void *b) { return *(int*)a - *(int*)b; }
int cmp_desc(const void *a, const void *b) { return *(int*)b - *(int*)a; }

int main() {
    int data[] = {7,5,9,0,3,1,6,2,4,8};
    qsort(data, 10, sizeof(int), cmp_asc);
    for (int i = 0; i < 10; i++) printf("%d ", data[i]);
}
```

### JavaScript (Node.js)
```javascript
const data = [7,5,9,0,3,1,6,2,4,8];

// 주의: sort() 는 기본 문자열 정렬! → 비교 함수 필수
data.sort((a, b) => a - b);          // 오름차순
data.sort((a, b) => b - a);          // 내림차순

const students = [['Alice',90],['Bob',80],['Cathy',90]];
students.sort((a, b) => b[1] - a[1] || a[0].localeCompare(b[0]));

function quickSort(arr) {
    if (arr.length <= 1) return arr;
    const [pivot, ...rest] = arr;
    const left = rest.filter(x => x <= pivot);
    const right = rest.filter(x => x > pivot);
    return [...quickSort(left), pivot, ...quickSort(right)];
}
console.log(quickSort(data));
```

### TypeScript
```typescript
const data: number[] = [7,5,9,0,3,1,6,2,4,8];
data.sort((a, b) => a - b);

interface Student { name: string; score: number; }
const students: Student[] = [
    {name:'Alice', score:90}, {name:'Bob', score:80}, {name:'Cathy', score:90}];
students.sort((a, b) => b.score - a.score || a.name.localeCompare(b.name));

function quickSort(arr: number[]): number[] {
    if (arr.length <= 1) return arr;
    const [pivot, ...rest] = arr;
    return [...quickSort(rest.filter(x => x <= pivot)), pivot,
            ...quickSort(rest.filter(x => x > pivot))];
}
```

### Kotlin
```kotlin
fun main() {
    val data = intArrayOf(7,5,9,0,3,1,6,2,4,8)
    data.sort()                             // 오름차순
    val desc = data.toTypedArray().apply { sortDescending() }
    
    data class Student(val name: String, val score: Int)
    val students = listOf(Student("Alice",90), Student("Bob",80), Student("Cathy",90))
    val sorted = students.sortedWith(compareByDescending<Student>{ it.score }.thenBy { it.name })
}

fun quickSort(a: IntArray, lo: Int = 0, hi: Int = a.size - 1) {
    if (lo < hi) {
        val pivot = a[hi]
        var i = lo - 1
        for (j in lo until hi) {
            if (a[j] <= pivot) { i++; val t = a[i]; a[i] = a[j]; a[j] = t }
        }
        val t = a[i+1]; a[i+1] = a[hi]; a[hi] = t
        quickSort(a, lo, i)
        quickSort(a, i+2, hi)
    }
}
```

### Swift
```swift
var data = [7,5,9,0,3,1,6,2,4,8]
data.sort()                        // 오름차순 (안정 보장 X)
data.sort(by: >)                   // 내림차순

let students = [("Alice",90),("Bob",80),("Cathy",90)]
let sorted = students.sorted { (a, b) -> Bool in
    if a.1 != b.1 { return a.1 > b.1 }
    return a.0 < b.0
}

func quickSort(_ arr: [Int]) -> [Int] {
    guard arr.count > 1 else { return arr }
    let pivot = arr.first!
    let rest = Array(arr.dropFirst())
    return quickSort(rest.filter { $0 <= pivot }) + [pivot] + quickSort(rest.filter { $0 > pivot })
}
```

### Go
```go
package main

import (
    "fmt"
    "sort"
)

type Student struct {
    Name  string
    Score int
}

func quickSort(a []int) {
    if len(a) < 2 { return }
    pivot := a[len(a)-1]
    i := -1
    for j := 0; j < len(a)-1; j++ {
        if a[j] <= pivot { i++; a[i], a[j] = a[j], a[i] }
    }
    a[i+1], a[len(a)-1] = a[len(a)-1], a[i+1]
    quickSort(a[:i+1])
    quickSort(a[i+2:])
}

func main() {
    data := []int{7,5,9,0,3,1,6,2,4,8}
    sort.Ints(data)
    sort.Sort(sort.Reverse(sort.IntSlice(data)))

    students := []Student{{"Alice",90},{"Bob",80},{"Cathy",90}}
    sort.Slice(students, func(i, j int) bool {
        if students[i].Score != students[j].Score { return students[i].Score > students[j].Score }
        return students[i].Name < students[j].Name
    })
    fmt.Println(students)
}
```

### Rust
```rust
fn quick_sort(a: &mut [i32]) {
    if a.len() < 2 { return; }
    let len = a.len();
    let pivot = a[len - 1];
    let mut i = 0;
    for j in 0..len - 1 {
        if a[j] <= pivot { a.swap(i, j); i += 1; }
    }
    a.swap(i, len - 1);
    let (left, right) = a.split_at_mut(i);
    quick_sort(left);
    quick_sort(&mut right[1..]);
}

fn main() {
    let mut data = vec![7,5,9,0,3,1,6,2,4,8];
    data.sort();                              // 안정 정렬 (Timsort variant)
    data.sort_unstable();                     // 불안정, 더 빠름
    data.sort_by(|a, b| b.cmp(a));            // 내림차순

    let mut students = vec![("Alice",90),("Bob",80),("Cathy",90)];
    students.sort_by(|a, b| b.1.cmp(&a.1).then(a.0.cmp(b.0)));
}
```

### C#
```csharp
using System;
using System.Linq;

public class SortDemo {
    public static void Main() {
        int[] data = {7,5,9,0,3,1,6,2,4,8};
        Array.Sort(data);                              // 오름차순 (introsort)
        Array.Sort(data, (a, b) => b.CompareTo(a));    // 내림차순

        var students = new[] {
            new {Name="Alice", Score=90},
            new {Name="Bob", Score=80},
            new {Name="Cathy", Score=90}};
        var sorted = students
            .OrderByDescending(s => s.Score)
            .ThenBy(s => s.Name)
            .ToArray();
    }

    static void QuickSort(int[] a, int lo, int hi) {
        if (lo < hi) {
            int pivot = a[hi], i = lo - 1;
            for (int j = lo; j < hi; j++)
                if (a[j] <= pivot) { i++; (a[i], a[j]) = (a[j], a[i]); }
            (a[i+1], a[hi]) = (a[hi], a[i+1]);
            QuickSort(a, lo, i);
            QuickSort(a, i+2, hi);
        }
    }
}
```

---

## 8. 변형 / 응용

### (a) k 번째 작은 수 (Quickselect)
퀵 정렬의 partition 만 반복. **O(N) 평균** — k 번째에만 관심 있을 때.

### (b) 부분 정렬 (Top-K)
힙 사용. 크기 K 의 min-heap 유지 → O(N log K).

### (c) 외부 정렬 (External Sort)
메모리에 다 안 들어가는 큰 데이터 → 디스크 + 병합 정렬.

### (d) 위상 정렬 — DAG 순서
정렬이라고 부르지만 그래프 알고리즘. [[../graph-theory/graph-theory]] 참조.

### (e) 다중 키 정렬
**stable sort** 활용 — 부수 키부터 먼저 정렬 → 주 키 정렬.

### (f) 사용자 정의 비교
- Python: `key=lambda x: (a, -b, c)`
- Java/JS/C++: 비교 함수 람다
- 주의: 비교 함수가 `transitive` (a<b, b<c → a<c) 안 되면 정렬 망가짐.

---

## 9. 함정 / 안티패턴

### 함정 1 — JavaScript `sort()` 가 문자열 정렬

```javascript
[10, 2, 1].sort();          // ['1', '10', '2'] ← 문자열로 비교!
[10, 2, 1].sort((a,b) => a-b);   // [1, 2, 10]   ← 올바름
```

### 함정 2 — 불안정 정렬의 부작용

C++ `std::sort` / Java `Arrays.sort(int[])` 는 **불안정**. 다중 키 정렬 시
주의:

```java
// 학생 점수 내림 + 이름 오름 정렬할 때
Arrays.sort(students, byName);       // 1차: 이름
Arrays.sort(students, byScoreDesc);  // 2차: 점수 — 이건 stable 정렬 필요!
// → 대신 한 번에 비교 함수로 처리하는 게 안전
```

Java 의 `Arrays.sort(Object[])` 는 안정 (Timsort).

### 함정 3 — Python `sort` 의 `reverse=True` + `key`

```python
data.sort(key=lambda x: x[1], reverse=True)
# == data.sort(key=lambda x: -x[1])

# 하지만 다중 키일 때 다름:
data.sort(key=lambda x: (-x[1], x[0]))   # 점수 내림 + 이름 오름 ← 올바름
```

### 함정 4 — N² 정렬에 N=10^5

선택/삽입/버블 정렬은 N=10000 부터 TLE. 거의 항상 내장 정렬 사용.

### 함정 5 — 사용자 정의 비교가 transitive 가 아님

```python
# 잘못 — 부동소수점 비교 / 동등 무시
cmp = lambda a, b: 1 if a > b + 0.001 else -1
# a=1.0, b=1.0005, c=1.001 → cmp(a,b)=-1, cmp(b,c)=-1, cmp(a,c)=1
# transitive 깨짐 → 정렬 결과 예측 불가
```

### 함정 6 — 메모리 / 시간 트레이드오프

계수 정렬 — 값 범위 K=10^9 면 메모리 폭발. K 작을 때만.

### 함정 7 — 정렬을 안 하고 이분 탐색

```python
# 잘못
data = [5, 1, 3, 2, 4]
bisect.bisect_left(data, 3)   # 정렬 안 돼서 결과 의미 없음

# 옳음
data.sort()
bisect.bisect_left(data, 3)
```

### 함정 8 — in-place vs 새 배열

- Python `data.sort()` — in-place, 반환 None.
- Python `sorted(data)` — 새 리스트.
- 헷갈리면 `data = data.sort()` 같은 실수.

---

## 10. 실전 풀이 절차 — 5 단계

1. **정렬이 필요한가?** — 입력이 정렬되어 있나 / 정렬하면 더 쉬워지나.
2. **정렬 기준 결정** — 단일 / 다중 키. 오름/내림.
3. **내장 정렬 활용** — 99% 의 경우 내장.
4. **안정성 필요?** — 다중 키 + 부수 기준 보존 시.
5. **시간/공간 한도 확인** — N=10^7 도 O(N log N) 1 초 이내.

---

## 11. 대표 문제 (이코테 Ch.06 + Ch.14)

### Q1. 위에서 아래로 (Ch.06)
큰 수 → 작은 수 정렬.

```python
n = int(input())
data = [int(input()) for _ in range(n)]
data.sort(reverse=True)
print(*data)
```

### Q2. 성적이 낮은 순서로 학생 출력하기 (Ch.06)
점수 오름차순.

```python
n = int(input())
students = []
for _ in range(n):
    name, score = input().split()
    students.append((name, int(score)))
students.sort(key=lambda x: x[1])
print(' '.join(s[0] for s in students))
```

### Q3. 두 배열의 원소 교체 (Ch.06)
A 의 최소값과 B 의 최댓값 K 번 swap → A 의 합 최대화.

```python
n, k = map(int, input().split())
a = list(map(int, input().split()))
b = list(map(int, input().split()))
a.sort()                # 작은 순
b.sort(reverse=True)    # 큰 순
for i in range(k):
    if a[i] < b[i]:
        a[i], b[i] = b[i], a[i]
    else:
        break
print(sum(a))
```

### Q4. 국영수 (Ch.14)
국어 내림 → 영어 오름 → 수학 내림 → 이름 오름 다중 정렬.

```python
n = int(input())
students = []
for _ in range(n):
    parts = input().split()
    students.append((parts[0], int(parts[1]), int(parts[2]), int(parts[3])))
students.sort(key=lambda x: (-x[1], x[2], -x[3], x[0]))
print('\n'.join(s[0] for s in students))
```

### Q5. 안테나 (Ch.14)
거리 합 최소 — **중앙값**.

```python
n = int(input())
data = sorted(list(map(int, input().split())))
print(data[(n - 1) // 2])
```
**포인트**: 정렬 후 중앙값이 답 (수학적 사실 — L1 norm minimizer).

### Q6. 실패율 (Ch.14, 카카오)
스테이지 N + 도달 사용자별 실패율 → 실패율 내림차순.

```python
def solution(N, stages):
    answer = []
    user_count = len(stages)
    for stage in range(1, N + 1):
        reached = sum(1 for s in stages if s >= stage)
        fail_at = sum(1 for s in stages if s == stage)
        rate = fail_at / reached if reached > 0 else 0
        answer.append((stage, rate))
    answer.sort(key=lambda x: (-x[1], x[0]))
    return [s for s, _ in answer]
```

### Q7. 카드 정렬하기 (Ch.14) — 우선순위 큐
N 묶음 카드를 합칠 때 비교 횟수 최소 → 매번 가장 작은 두 묶음 합치기 (허프만).

```python
import heapq
n = int(input())
heap = []
for _ in range(n):
    heapq.heappush(heap, int(input()))

result = 0
while len(heap) > 1:
    a = heapq.heappop(heap)
    b = heapq.heappop(heap)
    result += a + b
    heapq.heappush(heap, a + b)
print(result)
```

---

## 12. 연습 문제 (난이도별)

### 입문
- [백준 2750 수 정렬하기](https://www.acmicpc.net/problem/2750)
- [백준 2751 수 정렬하기 2](https://www.acmicpc.net/problem/2751) — N log N 필요
- [백준 10989 수 정렬하기 3](https://www.acmicpc.net/problem/10989) — 계수 정렬

### 중급
- [백준 1181 단어 정렬](https://www.acmicpc.net/problem/1181)
- [백준 11650 좌표 정렬하기](https://www.acmicpc.net/problem/11650)
- [백준 10814 나이순 정렬](https://www.acmicpc.net/problem/10814) — 안정 정렬
- [백준 11652 카드](https://www.acmicpc.net/problem/11652)

### 고급
- [백준 9012 괄호](https://www.acmicpc.net/problem/9012) — 정렬 X, 알고리즘 영역
- [백준 2470 두 용액](https://www.acmicpc.net/problem/2470) — 정렬 + 투 포인터
- [백준 13164 행복 유치원](https://www.acmicpc.net/problem/13164) — 정렬 + 그리디
- [LeetCode 215 Kth Largest](https://leetcode.com/problems/kth-largest-element-in-an-array/) — Quickselect

### 카카오 / 삼성
- [Programmers — H-Index](https://programmers.co.kr/learn/courses/30/lessons/42747)
- [Programmers — 가장 큰 수](https://programmers.co.kr/learn/courses/30/lessons/42746) — 커스텀 비교

---

## 13. 학습 자료

- 이코테 **Ch.06, 14** — 한국어 표준
- [동빈나 — 정렬 영상](https://www.youtube.com/watch?v=KGyK-pNvWos) (무료)
- [CLRS Ch.6-8](https://mitpress.mit.edu/9780262046305/) — Sorting 표준 학술
- [VisuAlgo — Sorting](https://visualgo.net/en/sorting) — 시각화 (한국어 지원)
- [Sorting Animations (toptal)](https://www.toptal.com/developers/sorting-algorithms)
- [Timsort 논문](https://github.com/python/cpython/blob/main/Objects/listsort.txt) — Python 의 Timsort 설명

---

## 14. 관련

- [[../greedy/greedy]] — 정렬 후 그리디
- [[../binary-search/binary-search]] — 정렬은 이분 탐색의 전제
- [[../graph-theory/graph-theory]] — 위상 정렬
- [[../algorithm|↑ algorithm 인덱스]]
