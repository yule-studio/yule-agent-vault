---
title: "다이나믹 프로그래밍 (Dynamic Programming)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T02:30:00+09:00
tags:
  - algorithm
  - dp
---

# 다이나믹 프로그래밍 (DP)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 수준 + 11 언어 (이코테 Ch.08/16) |

**[[../algorithm|↑ algorithm]]** · **[[../computer-science/computer-science|↑↑ computer-science]]**

---

## 1. 한 줄 정의

**큰 문제를 작은 부분 문제로 쪼개서 풀고, 그 답을 재사용** 해서 시간을
극적으로 줄이는 패러다임. 흔히 지수 시간 (`O(2^N)`) 문제를 다항식
(`O(N²)`) 으로 바꾼다.

---

## 2. DP 는 언제 쓰는가? — 시그널 인식

### (a) DP 의 2 필수 조건

1. **최적 부분 구조 (Optimal Substructure)** — 큰 문제의 최적해가 부분 문제
   의 최적해로 구성됨.
2. **중복 부분 문제 (Overlapping Subproblems)** — 같은 부분 문제가 여러 번
   호출됨. 한 번 계산하고 재사용.

두 조건 모두 만족 → DP. 부분 구조만 있고 중복 없으면 → 분할 정복 (예: 병합
정렬).

### (b) 문제 유형 키워드

- **"최댓값 / 최솟값"** + **"경우의 수"** (그리디 안 통하면)
- **"~ 가지 방법 / 몇 가지 경로 / 합 / 곱"**
- **"이전 단계 답 + 현재 결정 → 현재 답"** 점화식 가능
- **"피보나치 / 계단 오르기 / 동전 거스름돈 / 배낭"** — 표준 DP

### (c) DP vs 다른 알고리즘

| 차원 | DP | 그리디 | 완전 탐색 | 분할 정복 |
| --- | --- | --- | --- | --- |
| 최적 | ✅ 항상 | △ 정당성 필요 | ✅ 시간만 있다면 | ✅ 분리 가능할 때 |
| 시간 | `O(N²)` ~ `O(NM)` | `O(N log N)` | `O(2^N)` ~ `O(N!)` | `O(N log N)` |
| 신호 | 작은 → 큰 점화식 | 매번 best 선택 | 작은 N | 독립 부분 |

**판단**: 그리디 반례 → DP / 완탐 → 작은 N 이면 완탐, 큰 N 이면 DP.

### (d) DP 의 두 구현 방식

| 방식 | 방향 | 자료구조 | 장점 | 단점 |
| --- | --- | --- | --- | --- |
| **Top-down** (메모이제이션) | 큰 → 작은 (재귀) | 딕셔너리 / 배열 | 직관적 / 필요한 값만 계산 | 재귀 스택 / 함수 호출 오버헤드 |
| **Bottom-up** (타뷸레이션) | 작은 → 큰 (반복) | 배열 / 표 | 빠름 / 메모리 통제 | 점화식 / 순서 신중 |

---

## 3. 핵심 직관 — "이미 본 답 다시 안 풀기"

피보나치 `fib(n) = fib(n-1) + fib(n-2)`:

```
fib(5)
├── fib(4)
│   ├── fib(3)
│   │   ├── fib(2)
│   │   │   ├── fib(1) = 1
│   │   │   └── fib(0) = 0
│   │   └── fib(1) = 1
│   └── fib(2)   ← 이미 계산했음! 또?
└── fib(3)       ← 또?
```

**재귀**: `fib(50)` = 2^50 호출. 우주 끝나도 안 끝남.
**DP**: 같은 부분 문제를 한 번만 → `fib(50)` = 50 번 호출.

비유:
- **퀴즈 답안지** — 같은 문제 또 나오면 다시 풀지 말고 답안지 본다.
- **건물 짓기** — 1층 짓고 → 2층 → ... → 100층. 5층 짓는데 4층 또 짓지 않는다.

---

## 4. 동작 원리 — 예시 깊이

### (a) 피보나치 (가장 단순)

```
점화식:
  D[0] = 0
  D[1] = 1
  D[n] = D[n-1] + D[n-2]   (n >= 2)

계산 (Bottom-up, N=10):
  D[0]=0, D[1]=1
  D[2]=D[1]+D[0]=1
  D[3]=D[2]+D[1]=2
  D[4]=D[3]+D[2]=3
  D[5]=D[4]+D[3]=5
  D[6]=8, D[7]=13, D[8]=21, D[9]=34, D[10]=55
```

### (b) 1로 만들기 (이코테 Ch.08)

N 을 1 로 만드는 최소 연산 (-1, /2, /3, /5):

```
점화식:
  D[1] = 0
  D[n] = min(D[n-1], D[n//2], D[n//3], D[n//5]) + 1
         (가능한 연산만)

예: N = 26
  D[26] = min(D[25], D[13]) + 1
  D[25] = min(D[24], D[5]) + 1 = min(?, 2) + 1 = 3? (D[5]=2)
  D[13] = D[12] + 1 = ...
  ...

답: D[26] = 3 (26 → 25 → 5 → 1)
```

### (c) 개미 전사 (Ch.08) — "i-2 vs i-1"

식량 창고 N 개. 인접 약탈 X. 최대 약탈량.

```
점화식:
  D[0] = a[0]
  D[1] = max(a[0], a[1])
  D[i] = max(D[i-1], D[i-2] + a[i])   (i >= 2)

a = [1, 3, 1, 5]
D[0] = 1
D[1] = max(1, 3) = 3
D[2] = max(D[1], D[0]+1) = max(3, 2) = 3
D[3] = max(D[2], D[1]+5) = max(3, 8) = 8

답: 8 (a[1] + a[3] = 3 + 5)
```

### (d) 효율적인 화폐 구성 (Ch.08) — 0/1 vs Unbounded Knapsack 형태

동전 종류 + 금액 → 최소 동전 개수.

```
점화식:
  D[i] = min(D[i-coin]) + 1   (모든 coin 에 대해)

동전 = [2, 3], 금액 = 7:
  D[0] = 0
  D[1] = INF (불가능)
  D[2] = min(D[0])+1 = 1
  D[3] = min(D[1], D[0])+1 = 1
  D[4] = min(D[2], D[1])+1 = 2
  D[5] = min(D[3], D[2])+1 = 2
  D[6] = min(D[4], D[3])+1 = 2
  D[7] = min(D[5], D[4])+1 = 3
  
답: 3 (2+2+3)
```

---

## 5. 복잡도

| 변형 | 시간 | 공간 |
| --- | --- | --- |
| 1D DP (피보나치, 개미) | `O(N)` | `O(N)` → `O(1)` 최적화 가능 |
| 1D + 결정 K 개 | `O(N×K)` | `O(N)` 또는 `O(N×K)` |
| 2D DP (배낭, LCS) | `O(N×M)` | `O(N×M)` → `O(M)` 최적화 |
| 비트마스킹 DP | `O(2^N × N)` | `O(2^N)` |
| 트리 DP | `O(N)` | `O(N)` |

**메모리 최적화**: 마지막 1-2 행만 필요한 DP → `O(M)` 으로 축소.

---

## 6. 의사 코드

### Top-down (메모이제이션)
```
memo ← {}
function DP(state):
    if state in memo: return memo[state]
    if base_case(state): return base_value
    result ← combine(DP(sub_state1), DP(sub_state2), ...)
    memo[state] ← result
    return result
```

### Bottom-up (타뷸레이션)
```
D ← array of size N+1
D[base_index] ← base_value
for i from base+1 to N:
    D[i] ← recurrence(D[i-1], D[i-2], ..., input[i])
return D[N]
```

---

## 7. 언어별 구현 — 피보나치 + 1로 만들기 (11 언어)

### Python 3
```python
import sys
sys.setrecursionlimit(10**6)

# (1) Top-down 피보나치
memo = {}
def fib_topdown(n):
    if n in memo: return memo[n]
    if n <= 1: return n
    memo[n] = fib_topdown(n-1) + fib_topdown(n-2)
    return memo[n]

# (2) Bottom-up 피보나치 (공간 O(1))
def fib_bottomup(n):
    if n <= 1: return n
    a, b = 0, 1
    for _ in range(n - 1):
        a, b = b, a + b
    return b

# (3) 1로 만들기
def make_one(n):
    d = [0] * (n + 1)
    for i in range(2, n + 1):
        d[i] = d[i-1] + 1
        if i % 2 == 0: d[i] = min(d[i], d[i//2] + 1)
        if i % 3 == 0: d[i] = min(d[i], d[i//3] + 1)
        if i % 5 == 0: d[i] = min(d[i], d[i//5] + 1)
    return d[n]

print(fib_bottomup(50))   # 12586269025
print(make_one(26))        # 3
```

### Java
```java
import java.util.*;
public class DPDemo {
    static Map<Integer, Long> memo = new HashMap<>();

    static long fibTopdown(int n) {
        if (memo.containsKey(n)) return memo.get(n);
        if (n <= 1) return n;
        long r = fibTopdown(n-1) + fibTopdown(n-2);
        memo.put(n, r);
        return r;
    }

    static long fibBottomup(int n) {
        if (n <= 1) return n;
        long a = 0, b = 1;
        for (int i = 2; i <= n; i++) { long c = a + b; a = b; b = c; }
        return b;
    }

    static int makeOne(int n) {
        int[] d = new int[n + 1];
        for (int i = 2; i <= n; i++) {
            d[i] = d[i-1] + 1;
            if (i % 2 == 0) d[i] = Math.min(d[i], d[i/2] + 1);
            if (i % 3 == 0) d[i] = Math.min(d[i], d[i/3] + 1);
            if (i % 5 == 0) d[i] = Math.min(d[i], d[i/5] + 1);
        }
        return d[n];
    }
}
```

### C++
```cpp
#include <bits/stdc++.h>
using namespace std;

map<int, long long> memo;
long long fib_td(int n) {
    if (memo.count(n)) return memo[n];
    if (n <= 1) return n;
    return memo[n] = fib_td(n-1) + fib_td(n-2);
}

long long fib_bu(int n) {
    if (n <= 1) return n;
    long long a = 0, b = 1;
    for (int i = 2; i <= n; i++) { long long c = a + b; a = b; b = c; }
    return b;
}

int make_one(int n) {
    vector<int> d(n + 1, 0);
    for (int i = 2; i <= n; i++) {
        d[i] = d[i-1] + 1;
        if (i % 2 == 0) d[i] = min(d[i], d[i/2] + 1);
        if (i % 3 == 0) d[i] = min(d[i], d[i/3] + 1);
        if (i % 5 == 0) d[i] = min(d[i], d[i/5] + 1);
    }
    return d[n];
}
```

### C
```c
#include <stdio.h>
#include <stdlib.h>

long long fib_bu(int n) {
    if (n <= 1) return n;
    long long a = 0, b = 1;
    for (int i = 2; i <= n; i++) { long long c = a + b; a = b; b = c; }
    return b;
}

int make_one(int n) {
    int *d = calloc(n + 1, sizeof(int));
    for (int i = 2; i <= n; i++) {
        d[i] = d[i-1] + 1;
        if (i % 2 == 0 && d[i/2] + 1 < d[i]) d[i] = d[i/2] + 1;
        if (i % 3 == 0 && d[i/3] + 1 < d[i]) d[i] = d[i/3] + 1;
        if (i % 5 == 0 && d[i/5] + 1 < d[i]) d[i] = d[i/5] + 1;
    }
    int ans = d[n];
    free(d);
    return ans;
}

int main() {
    printf("%lld\n", fib_bu(50));
    printf("%d\n", make_one(26));
}
```

### JavaScript (Node.js)
```javascript
const memo = new Map();
function fibTD(n) {
    if (memo.has(n)) return memo.get(n);
    if (n <= 1) return BigInt(n);
    const r = fibTD(n-1) + fibTD(n-2);
    memo.set(n, r);
    return r;
}

function fibBU(n) {
    if (n <= 1) return BigInt(n);
    let [a, b] = [0n, 1n];
    for (let i = 2; i <= n; i++) [a, b] = [b, a + b];
    return b;
}

function makeOne(n) {
    const d = new Array(n + 1).fill(0);
    for (let i = 2; i <= n; i++) {
        d[i] = d[i-1] + 1;
        if (i % 2 === 0) d[i] = Math.min(d[i], d[i/2] + 1);
        if (i % 3 === 0) d[i] = Math.min(d[i], Math.floor(i/3) === i/3 ? d[i/3] + 1 : d[i]);
        if (i % 5 === 0) d[i] = Math.min(d[i], d[i/5] + 1);
    }
    return d[n];
}

console.log(fibBU(50).toString());
console.log(makeOne(26));
```

### TypeScript
```typescript
function fibBU(n: number): bigint {
    if (n <= 1) return BigInt(n);
    let [a, b] = [0n, 1n];
    for (let i = 2; i <= n; i++) [a, b] = [b, a + b];
    return b;
}

function makeOne(n: number): number {
    const d: number[] = new Array(n + 1).fill(0);
    for (let i = 2; i <= n; i++) {
        d[i] = d[i-1] + 1;
        if (i % 2 === 0) d[i] = Math.min(d[i], d[i/2] + 1);
        if (i % 3 === 0) d[i] = Math.min(d[i], d[i/3] + 1);
        if (i % 5 === 0) d[i] = Math.min(d[i], d[i/5] + 1);
    }
    return d[n];
}
```

### Kotlin
```kotlin
fun fibBU(n: Int): Long {
    if (n <= 1) return n.toLong()
    var a = 0L; var b = 1L
    for (i in 2..n) { val c = a + b; a = b; b = c }
    return b
}

fun makeOne(n: Int): Int {
    val d = IntArray(n + 1)
    for (i in 2..n) {
        d[i] = d[i-1] + 1
        if (i % 2 == 0) d[i] = minOf(d[i], d[i/2] + 1)
        if (i % 3 == 0) d[i] = minOf(d[i], d[i/3] + 1)
        if (i % 5 == 0) d[i] = minOf(d[i], d[i/5] + 1)
    }
    return d[n]
}
```

### Swift
```swift
func fibBU(_ n: Int) -> Int {
    if n <= 1 { return n }
    var a = 0, b = 1
    for _ in 2...n { (a, b) = (b, a + b) }
    return b
}

func makeOne(_ n: Int) -> Int {
    var d = Array(repeating: 0, count: n + 1)
    for i in 2...n {
        d[i] = d[i-1] + 1
        if i % 2 == 0 { d[i] = min(d[i], d[i/2] + 1) }
        if i % 3 == 0 { d[i] = min(d[i], d[i/3] + 1) }
        if i % 5 == 0 { d[i] = min(d[i], d[i/5] + 1) }
    }
    return d[n]
}
```

### Go
```go
package main

import "fmt"

func fibBU(n int) int64 {
    if n <= 1 { return int64(n) }
    var a, b int64 = 0, 1
    for i := 2; i <= n; i++ {
        a, b = b, a + b
    }
    return b
}

func makeOne(n int) int {
    d := make([]int, n + 1)
    for i := 2; i <= n; i++ {
        d[i] = d[i-1] + 1
        if i % 2 == 0 && d[i/2] + 1 < d[i] { d[i] = d[i/2] + 1 }
        if i % 3 == 0 && d[i/3] + 1 < d[i] { d[i] = d[i/3] + 1 }
        if i % 5 == 0 && d[i/5] + 1 < d[i] { d[i] = d[i/5] + 1 }
    }
    return d[n]
}

func main() {
    fmt.Println(fibBU(50))
    fmt.Println(makeOne(26))
}
```

### Rust
```rust
fn fib_bu(n: u32) -> u64 {
    if n <= 1 { return n as u64; }
    let (mut a, mut b): (u64, u64) = (0, 1);
    for _ in 2..=n { let c = a + b; a = b; b = c; }
    b
}

fn make_one(n: usize) -> i32 {
    let mut d = vec![0; n + 1];
    for i in 2..=n {
        d[i] = d[i-1] + 1;
        if i % 2 == 0 { d[i] = d[i].min(d[i/2] + 1); }
        if i % 3 == 0 { d[i] = d[i].min(d[i/3] + 1); }
        if i % 5 == 0 { d[i] = d[i].min(d[i/5] + 1); }
    }
    d[n]
}

fn main() {
    println!("{}", fib_bu(50));
    println!("{}", make_one(26));
}
```

### C#
```csharp
using System;
public class DPDemo {
    public static long FibBU(int n) {
        if (n <= 1) return n;
        long a = 0, b = 1;
        for (int i = 2; i <= n; i++) { long c = a + b; a = b; b = c; }
        return b;
    }

    public static int MakeOne(int n) {
        int[] d = new int[n + 1];
        for (int i = 2; i <= n; i++) {
            d[i] = d[i-1] + 1;
            if (i % 2 == 0) d[i] = Math.Min(d[i], d[i/2] + 1);
            if (i % 3 == 0) d[i] = Math.Min(d[i], d[i/3] + 1);
            if (i % 5 == 0) d[i] = Math.Min(d[i], d[i/5] + 1);
        }
        return d[n];
    }
}
```

---

## 8. 변형 / 응용 — DP 의 주요 패턴

### (a) 1D — 선형 DP
- 피보나치 / 계단 오르기 / 개미 전사 / 정수 삼각형

### (b) 2D — 격자 / 행렬
- 경로 수 / 최대값 경로 / 편집 거리 / LCS (Longest Common Subsequence)

### (c) 배낭 (Knapsack)
- **0/1 Knapsack** — 각 아이템 1 번만. 점화식 `D[i][w] = max(D[i-1][w], D[i-1][w-weight]+value)`.
- **Unbounded Knapsack** — 무제한 사용. 동전 거스름돈.
- **Bounded Knapsack** — 각 K 번까지. 0/1 + binary decomposition.

### (d) LIS (가장 긴 증가 부분 수열)
- O(N²) DP
- O(N log N) — 이분 탐색 결합

### (e) LCS / 편집 거리
- 두 문자열 간 최장 공통 부분 / 변환 비용

### (f) 비트마스킹 DP
- N ≤ 20 정도에서 부분집합을 비트마스크로 인덱싱
- 외판원 (TSP) / 부분 집합 합

### (g) 트리 DP
- 트리에서 부분 트리의 답을 누적

### (h) 구간 DP
- `D[i][j]` = 구간 [i, j] 의 답. 행렬 곱셈 순서 / 팰린드롬 / 회문 분할

---

## 9. 함정 / 안티패턴

### 함정 1 — 점화식이 안 떠오름
- **표 그리기** — N=1, 2, 3, 4 손풀이.
- **마지막 결정** 부터 역추적 — "마지막에 뭘 선택했나?" 가 차원.

### 함정 2 — DP vs 그리디 혼동
- 작은 케이스 1-5 로 그리디 시도 → 반례 보이면 DP.
- 화폐 종류 [1, 3, 4] / 거스름돈 6 — 그리디 3개 vs DP 2개.

### 함정 3 — 메모이제이션 깊이 (재귀)
Python 기본 1000 → `sys.setrecursionlimit(10**6)` 또는 Bottom-up.

### 함정 4 — Top-down 의 hidden 비용
함수 호출 / 해시 lookup 으로 Bottom-up 보다 2-5배 느림. 시간 빠듯하면 Bottom-up.

### 함정 5 — 메모리 폭발
N=10^4 × N=10^4 = 10^8 셀 → 메모리 초과. **공간 최적화** — 마지막 행만 유지.

### 함정 6 — 점화식의 base case 누락
`D[0]`, `D[1]` 만 채우고 시작. 누락하면 처음부터 wrong.

### 함정 7 — 비결정값 INF / 음수
`min` 일 때 초기값 = `INF` (very large), `max` 일 때 = `-INF`.

### 함정 8 — 0/1 vs Unbounded Knapsack 순서
```python
# 0/1 (각 아이템 1 번) — w 를 큰 순으로
for item in items:
    for w in range(W, weight-1, -1):
        d[w] = max(d[w], d[w-weight] + value)

# Unbounded (무제한) — w 를 작은 순으로
for item in items:
    for w in range(weight, W+1):
        d[w] = max(d[w], d[w-weight] + value)
```

---

## 10. 실전 풀이 절차 — 7 단계

1. **부분 구조 식별** — 큰 문제를 작은 부분으로 쪼갤 수 있나?
2. **상태 정의** — `D[i]` 또는 `D[i][j]` 가 무엇을 의미?
3. **점화식** — 이전 상태로 현재 어떻게 표현?
4. **Base case** — 가장 작은 입력에서 답.
5. **계산 순서** — Top-down 재귀 / Bottom-up 반복.
6. **구현** — 배열 / 딕셔너리.
7. **답 추출** — `D[N]` / `min(D)` / 역추적.

---

## 11. 대표 문제 (이코테 Ch.08 + Ch.16)

### Q1. 1로 만들기 (Ch.08)
위 코드 참조.

### Q2. 개미 전사 (Ch.08)

```python
n = int(input())
a = list(map(int, input().split()))
d = [0] * n
d[0] = a[0]
d[1] = max(a[0], a[1])
for i in range(2, n):
    d[i] = max(d[i-1], d[i-2] + a[i])
print(d[-1])
```

### Q3. 바닥 공사 (Ch.08)
1×N 바닥을 1×2 또는 2×1 또는 1×1 타일로 채우는 경우.

```python
n = int(input())
d = [0] * (n + 1)
d[1] = 1; d[2] = 3
for i in range(3, n + 1):
    d[i] = (d[i-1] + 2 * d[i-2]) % 796796
print(d[n])
```
**점화식**: 마지막 칸 = 1×1 (D[i-1]) + 2×1 (D[i-2]) + 1×2 (D[i-2]).

### Q4. 효율적인 화폐 구성 (Ch.08) — Unbounded Knapsack

```python
n, m = map(int, input().split())
coins = [int(input()) for _ in range(n)]
INF = 10001
d = [INF] * (m + 1)
d[0] = 0
for c in coins:
    for w in range(c, m + 1):
        d[w] = min(d[w], d[w-c] + 1)
print(d[m] if d[m] != INF else -1)
```

### Q5. 금광 (Ch.16) — 2D DP

```python
for _ in range(int(input())):
    n, m = map(int, input().split())
    arr = list(map(int, input().split()))
    d = [[0]*m for _ in range(n)]
    for i in range(n):
        d[i][0] = arr[i*m]
    for j in range(1, m):
        for i in range(n):
            left_up = d[i-1][j-1] if i > 0 else 0
            left = d[i][j-1]
            left_down = d[i+1][j-1] if i < n-1 else 0
            d[i][j] = arr[i*m + j] + max(left_up, left, left_down)
    print(max(d[i][m-1] for i in range(n)))
```

### Q6. 정수 삼각형 (Ch.16)

```python
n = int(input())
tri = [list(map(int, input().split())) for _ in range(n)]
for i in range(1, n):
    for j in range(i + 1):
        up_left = tri[i-1][j-1] if j > 0 else 0
        up_right = tri[i-1][j] if j < i else 0
        tri[i][j] += max(up_left, up_right)
print(max(tri[-1]))
```

### Q7. 퇴사 (Ch.16) — 결정 DP
N+1 일 후 일정. 각 날 일하면 P 일 후 보수. 최대화.

```python
n = int(input())
t = []; p = []
for _ in range(n):
    a, b = map(int, input().split())
    t.append(a); p.append(b)

d = [0] * (n + 1)
for i in range(n - 1, -1, -1):
    if i + t[i] > n:
        d[i] = d[i+1]
    else:
        d[i] = max(d[i+1], p[i] + d[i + t[i]])
print(d[0])
```

### Q8. 편집 거리 (Ch.16, Levenshtein Distance)

```python
def edit_distance(s1, s2):
    n, m = len(s1), len(s2)
    d = [[0]*(m+1) for _ in range(n+1)]
    for i in range(n+1): d[i][0] = i
    for j in range(m+1): d[0][j] = j
    for i in range(1, n+1):
        for j in range(1, m+1):
            if s1[i-1] == s2[j-1]:
                d[i][j] = d[i-1][j-1]
            else:
                d[i][j] = 1 + min(d[i-1][j], d[i][j-1], d[i-1][j-1])
    return d[n][m]

print(edit_distance("sunday", "saturday"))   # 3
```

---

## 12. 연습 문제 (난이도별)

### 입문
- [백준 1003 피보나치 함수](https://www.acmicpc.net/problem/1003)
- [백준 9095 1, 2, 3 더하기](https://www.acmicpc.net/problem/9095)
- [백준 11726 2×n 타일링](https://www.acmicpc.net/problem/11726)

### 중급
- [백준 1932 정수 삼각형](https://www.acmicpc.net/problem/1932)
- [백준 11052 카드 구매하기](https://www.acmicpc.net/problem/11052) — Unbounded Knapsack
- [백준 11053 LIS](https://www.acmicpc.net/problem/11053)
- [백준 12865 평범한 배낭](https://www.acmicpc.net/problem/12865) — 0/1 Knapsack

### 고급
- [백준 12015 LIS 2](https://www.acmicpc.net/problem/12015) — 이분 탐색 결합
- [백준 9251 LCS](https://www.acmicpc.net/problem/9251)
- [백준 7579 앱](https://www.acmicpc.net/problem/7579) — Knapsack 변형
- [백준 2098 외판원 순회](https://www.acmicpc.net/problem/2098) — 비트마스킹 DP

---

## 13. 학습 자료

- 이코테 **Ch.08, 16**
- [동빈나 — DP 영상](https://www.youtube.com/watch?v=5Lu34WIx2Us) (무료)
- [CLRS Ch.15](https://mitpress.mit.edu/9780262046305/) — Dynamic Programming 표준
- [VisuAlgo — DP](https://visualgo.net/en/recursion) — 시각화
- [LeetCode DP Patterns](https://leetcode.com/discuss/general-discussion/458695/dynamic-programming-patterns) — 패턴 모음
- [DP 패턴 정리 (LeetCode)](https://leetcode.com/discuss/study-guide/1437879/) — 영문

---

## 14. 관련

- [[../greedy/greedy]] — 그리디 안 통할 때 DP
- [[../binary-search/binary-search]] — LIS 등에서 결합
- [[../algorithm|↑ algorithm 인덱스]]
