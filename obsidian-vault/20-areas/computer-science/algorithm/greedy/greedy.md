---
title: "그리디 (Greedy Algorithm)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T00:00:00+09:00
tags:
  - algorithm
  - greedy
---

# 그리디 (Greedy Algorithm)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.2.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 수준 재작성 + 11 언어 구현 |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 최초 — 이코테 Ch.03/11 매핑 |

**[[../algorithm|↑ algorithm 인덱스]]**

---

## 1. 한 줄 정의

**현재 단계에서 가장 좋아 보이는 선택** 을 매번 하고, 그 선택을 **되돌리지
않는** 알고리즘 패러다임. 이 "근시안적" 선택이 **결국 전체 최적해** 가
되는 문제에서만 정답을 낸다.

---

## 2. 그리디는 언제 쓰는가? — 시그널 인식

### (a) 문제 유형 키워드

문제에 다음 단어가 나오면 그리디를 먼저 의심한다:

- **"최소 / 최대"** + 단순한 조건 (예: 최소 동전 개수, 최대 회의 수)
- **"가장 빨리 / 가장 적게 / 가장 많이"**
- **"k 번만 / 1 번만 / 한 번씩"** 식의 제한된 선택
- **정렬 후 처리** 가 자연스러운 문제

### (b) DP 와 구분하는 신호

DP 와 그리디는 둘 다 최적화 문제에 쓰이지만 결정 기준이 다르다:

| 차원 | 그리디 | DP |
| --- | --- | --- |
| 결정 시점 | 현재만 보고 선택 | 모든 가능한 경우 비교 |
| 되돌림 | 없음 | 있음 (memoization / table) |
| 정당성 | "교환해도 손해 없음" 증명 필요 | 점화식이 항상 성립 |
| 복잡도 | 보통 `O(N log N)` (정렬 + 1 패스) | 보통 `O(N^2)` ~ `O(NM)` |
| 신호 | 정렬 후 매번 best 선택 가능 | 부분 문제로 쪼개야 함 |

**판단 절차**:
1. 우선 작은 예시로 그리디 시도.
2. 반례 (counter-example) 가 안 보이면 → 그리디 시도.
3. 반례가 나오면 → DP / 완전탐색 / 백트래킹.

### (c) 정당성 증명 패턴

그리디가 최적임을 보이는 표준 패턴 2 가지:

#### 교환 인자 (Exchange Argument)
"그리디가 고른 답과 최적해를 비교 → 그리디 답으로 바꿔도 답이 나빠지지 않음" 을
보인다. 예: 회의실 배정에서 "끝나는 시간이 빠른" 회의를 고르는 그리디가 최적
인 이유 — 다른 답을 골라도 결국 그 자리에 끝나는 시간 빠른 회의를 끼울 수
있음.

#### 귀납법
1 단계가 최적 → k 단계 최적 가정 → k+1 단계도 최적 임을 보인다.

### (d) 그리디가 안 되는 대표 반례

**거스름돈 문제 — 동전이 배수 관계 아닐 때**:
- 동전: 1, 3, 4
- 거스름: 6
- 그리디: 4 + 1 + 1 = **3 개**
- 최적: 3 + 3 = **2 개**

→ 동전 단위가 서로 배수 관계 (예: 1, 5, 10, 50, 100, 500) 일 때만 그리디 최적.

---

## 3. 핵심 직관 — "쇼핑 카트 채우기"

배낭에 물건을 담는데 무게 한도가 있다. 한 단위 무게당 가격이 가장 높은 물건
부터 차례로 담으면 가치가 최대. **이게 그리디**.

비유:
- **연봉 협상** — 한 협상에서 최대 인상을 받고, 다음 협상은 그 결과 위에서
  다시 최대 인상. (실제로는 종합 패키지를 봐야 하니 그리디가 항상 최적은
  아님 — DP / 협상 이론 영역)
- **고속도로 휴게소** — 다음 휴게소가 너무 멀어 못 갈 것 같으면 지금 들른다.

---

## 4. 동작 원리 — 예시 데이터로 깊이 들여다보기

### 예시 1 — 거스름돈 (그리디 OK 케이스)

```
입력: 거스름돈 1260원, 동전 [500, 100, 50, 10]
```

```
Step 1: 동전 큰 순 정렬 → [500, 100, 50, 10]
Step 2: 500원 → 1260 // 500 = 2 개. 남은 260원.
Step 3: 100원 → 260 // 100 = 2 개. 남은 60원.
Step 4: 50원  → 60 // 50 = 1 개. 남은 10원.
Step 5: 10원  → 10 // 10 = 1 개. 남은 0원.

결과: 2 + 2 + 1 + 1 = 6 개.
```

**왜 큰 동전부터?** 작은 동전 N 개를 더 큰 동전 1 개로 대체 가능한 한
"개수"가 줄어든다. 동전 단위 배수 관계라서 항상 성립.

### 예시 2 — 회의실 배정 (Activity Selection)

회의실 1 개에 회의 N 개. 시간 안 겹치게 최대 몇 개 잡을 수 있나?

```
입력:
  회의 1: [1, 4]
  회의 2: [3, 5]
  회의 3: [0, 6]
  회의 4: [5, 7]
  회의 5: [8, 9]
  회의 6: [5, 9]
```

**그리디 전략**: "끝나는 시간이 가장 빠른" 회의를 매번 선택.

```
정렬 (끝나는 시간 오름차순):
  [1, 4] [3, 5] [0, 6] [5, 7] [5, 9] [8, 9]

Step 1: [1, 4] 선택. 끝나는 시간 = 4.
Step 2: [3, 5] → 시작 3 < 4. 건너뜀.
Step 3: [0, 6] → 시작 0 < 4. 건너뜀.
Step 4: [5, 7] → 시작 5 >= 4. 선택. 끝 = 7.
Step 5: [5, 9] → 시작 5 < 7. 건너뜀.
Step 6: [8, 9] → 시작 8 >= 7. 선택. 끝 = 9.

결과: 3 개 ([1,4] [5,7] [8,9])
```

**왜 "끝나는 시간 빠른" 게 최적?**
- 끝나는 시간이 빠를수록 남은 시간이 길어진다 = 다음 회의 끼울 여유.
- 교환 인자: 다른 회의를 첫 번째로 골랐을 때, 그 회의 대신 "끝나는 시간이
  가장 빠른" 회의로 바꿔도 답의 개수는 줄지 않는다.

---

## 5. 복잡도 분석

| 변형 | 시간 | 공간 | 설명 |
| --- | --- | --- | --- |
| 거스름돈 (정렬된 동전) | `O(N)` | `O(1)` | N=동전 종류. 단일 패스 |
| 회의실 배정 | `O(N log N)` | `O(N)` | 정렬 1 회 + 1 패스 |
| Huffman 코딩 | `O(N log N)` | `O(N)` | 우선순위 큐 |
| Kruskal MST | `O(E log E)` | `O(V+E)` | 간선 정렬 + Union-Find |
| Dijkstra | `O((V+E) log V)` | `O(V+E)` | min-heap |

대부분 **정렬이 병목** — `O(N log N)`.

---

## 6. 의사 코드 (일반 패턴)

```
function GREEDY(items, criterion):
    sort items by criterion        # O(N log N)
    result ← empty list
    for item in items:              # O(N)
        if accept(item, result):
            result.add(item)
    return result

function accept(item, current):
    # 문제별 조건. 회의실이면 "현재 마지막 회의 끝 시간 <= item 시작 시간"
    ...
```

---

## 7. 언어별 구현 — 거스름돈 + 회의실 배정 (11 언어)

### Python 3
```python
# (1) 거스름돈
def coin_change_greedy(amount: int, coins: list[int]) -> int:
    coins = sorted(coins, reverse=True)
    count = 0
    for coin in coins:
        count += amount // coin
        amount %= coin
    return count

# (2) 회의실 배정
def max_meetings(meetings: list[tuple[int, int]]) -> int:
    meetings.sort(key=lambda m: m[1])  # 끝나는 시간 오름차순
    count, end = 0, 0
    for start, finish in meetings:
        if start >= end:
            count += 1
            end = finish
    return count

print(coin_change_greedy(1260, [500, 100, 50, 10]))           # 6
print(max_meetings([(1,4),(3,5),(0,6),(5,7),(8,9),(5,9)]))    # 3
```

### Java
```java
import java.util.*;

public class Greedy {
    static int coinChangeGreedy(int amount, int[] coins) {
        Integer[] cs = Arrays.stream(coins).boxed().toArray(Integer[]::new);
        Arrays.sort(cs, Collections.reverseOrder());
        int count = 0;
        for (int c : cs) { count += amount / c; amount %= c; }
        return count;
    }

    static int maxMeetings(int[][] meetings) {
        Arrays.sort(meetings, (a, b) -> a[1] - b[1]);  // 끝나는 시간
        int count = 0, end = 0;
        for (int[] m : meetings) {
            if (m[0] >= end) { count++; end = m[1]; }
        }
        return count;
    }

    public static void main(String[] args) {
        System.out.println(coinChangeGreedy(1260, new int[]{500, 100, 50, 10})); // 6
        int[][] meetings = {{1,4},{3,5},{0,6},{5,7},{8,9},{5,9}};
        System.out.println(maxMeetings(meetings));  // 3
    }
}
```

### C++
```cpp
#include <bits/stdc++.h>
using namespace std;

int coinChangeGreedy(int amount, vector<int> coins) {
    sort(coins.rbegin(), coins.rend());
    int count = 0;
    for (int c : coins) { count += amount / c; amount %= c; }
    return count;
}

int maxMeetings(vector<pair<int,int>> meetings) {
    sort(meetings.begin(), meetings.end(),
         [](auto& a, auto& b){ return a.second < b.second; });
    int count = 0, end = 0;
    for (auto& [s, f] : meetings) {
        if (s >= end) { count++; end = f; }
    }
    return count;
}

int main() {
    cout << coinChangeGreedy(1260, {500, 100, 50, 10}) << "\n";   // 6
    cout << maxMeetings({{1,4},{3,5},{0,6},{5,7},{8,9},{5,9}}) << "\n"; // 3
}
```

### C
```c
#include <stdio.h>
#include <stdlib.h>

int cmp_desc(const void *a, const void *b) { return *(int*)b - *(int*)a; }
int cmp_finish(const void *a, const void *b) {
    int *p = (int*)a, *q = (int*)b;
    return p[1] - q[1];   // 끝나는 시간 오름차순
}

int coin_change_greedy(int amount, int coins[], int n) {
    qsort(coins, n, sizeof(int), cmp_desc);
    int count = 0;
    for (int i = 0; i < n; i++) {
        count += amount / coins[i];
        amount %= coins[i];
    }
    return count;
}

int max_meetings(int meetings[][2], int n) {
    qsort(meetings, n, sizeof(meetings[0]), cmp_finish);
    int count = 0, end = 0;
    for (int i = 0; i < n; i++) {
        if (meetings[i][0] >= end) {
            count++;
            end = meetings[i][1];
        }
    }
    return count;
}

int main() {
    int coins[] = {500, 100, 50, 10};
    printf("%d\n", coin_change_greedy(1260, coins, 4));     // 6
    int meetings[6][2] = {{1,4},{3,5},{0,6},{5,7},{8,9},{5,9}};
    printf("%d\n", max_meetings(meetings, 6));               // 3
    return 0;
}
```

### JavaScript (Node.js)
```javascript
function coinChangeGreedy(amount, coins) {
    coins = [...coins].sort((a, b) => b - a);
    let count = 0;
    for (const c of coins) {
        count += Math.floor(amount / c);
        amount %= c;
    }
    return count;
}

function maxMeetings(meetings) {
    meetings = [...meetings].sort((a, b) => a[1] - b[1]);
    let count = 0, end = 0;
    for (const [s, f] of meetings) {
        if (s >= end) { count++; end = f; }
    }
    return count;
}

console.log(coinChangeGreedy(1260, [500, 100, 50, 10]));            // 6
console.log(maxMeetings([[1,4],[3,5],[0,6],[5,7],[8,9],[5,9]]));    // 3
```

### TypeScript
```typescript
function coinChangeGreedy(amount: number, coins: number[]): number {
    coins = [...coins].sort((a, b) => b - a);
    let count = 0;
    for (const c of coins) {
        count += Math.floor(amount / c);
        amount %= c;
    }
    return count;
}

function maxMeetings(meetings: [number, number][]): number {
    meetings = [...meetings].sort((a, b) => a[1] - b[1]);
    let count = 0, end = 0;
    for (const [s, f] of meetings) {
        if (s >= end) { count++; end = f; }
    }
    return count;
}

console.log(coinChangeGreedy(1260, [500, 100, 50, 10]));
console.log(maxMeetings([[1,4],[3,5],[0,6],[5,7],[8,9],[5,9]]));
```

### Kotlin
```kotlin
fun coinChangeGreedy(amount: Int, coins: IntArray): Int {
    val sorted = coins.sortedDescending()
    var remaining = amount
    var count = 0
    for (c in sorted) {
        count += remaining / c
        remaining %= c
    }
    return count
}

fun maxMeetings(meetings: List<Pair<Int, Int>>): Int {
    val sorted = meetings.sortedBy { it.second }
    var count = 0
    var end = 0
    for ((s, f) in sorted) {
        if (s >= end) { count++; end = f }
    }
    return count
}

fun main() {
    println(coinChangeGreedy(1260, intArrayOf(500, 100, 50, 10)))  // 6
    println(maxMeetings(listOf(1 to 4, 3 to 5, 0 to 6, 5 to 7, 8 to 9, 5 to 9))) // 3
}
```

### Swift
```swift
func coinChangeGreedy(_ amount: Int, _ coins: [Int]) -> Int {
    var amt = amount
    var count = 0
    for c in coins.sorted(by: >) {
        count += amt / c
        amt %= c
    }
    return count
}

func maxMeetings(_ meetings: [(Int, Int)]) -> Int {
    let sorted = meetings.sorted { $0.1 < $1.1 }
    var count = 0, end = 0
    for (s, f) in sorted {
        if s >= end { count += 1; end = f }
    }
    return count
}

print(coinChangeGreedy(1260, [500, 100, 50, 10]))                     // 6
print(maxMeetings([(1,4),(3,5),(0,6),(5,7),(8,9),(5,9)]))             // 3
```

### Go
```go
package main

import (
    "fmt"
    "sort"
)

func coinChangeGreedy(amount int, coins []int) int {
    sort.Sort(sort.Reverse(sort.IntSlice(coins)))
    count := 0
    for _, c := range coins {
        count += amount / c
        amount %= c
    }
    return count
}

func maxMeetings(meetings [][2]int) int {
    sort.Slice(meetings, func(i, j int) bool {
        return meetings[i][1] < meetings[j][1]
    })
    count, end := 0, 0
    for _, m := range meetings {
        if m[0] >= end {
            count++
            end = m[1]
        }
    }
    return count
}

func main() {
    fmt.Println(coinChangeGreedy(1260, []int{500, 100, 50, 10}))         // 6
    fmt.Println(maxMeetings([][2]int{{1,4},{3,5},{0,6},{5,7},{8,9},{5,9}})) // 3
}
```

### Rust
```rust
fn coin_change_greedy(mut amount: u32, mut coins: Vec<u32>) -> u32 {
    coins.sort_unstable_by(|a, b| b.cmp(a));
    let mut count = 0;
    for c in coins {
        count += amount / c;
        amount %= c;
    }
    count
}

fn max_meetings(mut meetings: Vec<(u32, u32)>) -> u32 {
    meetings.sort_by_key(|m| m.1);
    let mut count = 0;
    let mut end = 0;
    for (s, f) in meetings {
        if s >= end {
            count += 1;
            end = f;
        }
    }
    count
}

fn main() {
    println!("{}", coin_change_greedy(1260, vec![500, 100, 50, 10]));        // 6
    let m = vec![(1,4),(3,5),(0,6),(5,7),(8,9),(5,9)];
    println!("{}", max_meetings(m));                                           // 3
}
```

### C#
```csharp
using System;
using System.Linq;
using System.Collections.Generic;

public class Greedy {
    public static int CoinChangeGreedy(int amount, int[] coins) {
        var sorted = coins.OrderByDescending(c => c);
        int count = 0;
        foreach (var c in sorted) {
            count += amount / c;
            amount %= c;
        }
        return count;
    }

    public static int MaxMeetings((int, int)[] meetings) {
        var sorted = meetings.OrderBy(m => m.Item2).ToArray();
        int count = 0, end = 0;
        foreach (var (s, f) in sorted) {
            if (s >= end) { count++; end = f; }
        }
        return count;
    }

    public static void Main() {
        Console.WriteLine(CoinChangeGreedy(1260, new[] {500, 100, 50, 10}));  // 6
        Console.WriteLine(MaxMeetings(new[] {(1,4),(3,5),(0,6),(5,7),(8,9),(5,9)})); // 3
    }
}
```

---

## 8. 변형 / 응용 — 같은 그리디가 다른 옷을 입은 모습

### (a) Activity Selection (회의실 배정) — 위에서 본 그것
끝나는 시간 정렬 + 가능한 회의 선택.

### (b) Fractional Knapsack (부분 배낭)
물건을 **쪼갤 수 있는** 배낭 문제. 단위 가치 (가격 / 무게) 가 높은 순으로
담는다. **0/1 Knapsack 은 DP 영역** (쪼갤 수 없음).

### (c) Huffman Coding (허프만 인코딩)
빈도가 낮은 문자 2 개를 합쳐 트리 만들기. min-heap 사용.

### (d) Job Sequencing with Deadlines
마감일 + 보상이 있는 작업 N 개. 보상 최대화. **보상 큰 순** 정렬 + 마감일
직전 비어있는 슬롯에 배치.

### (e) Minimum Spanning Tree (Kruskal)
가중치 작은 간선 순으로 추가. 사이클 만들지 않으면 채택. → Union-Find.
[[../graph-theory/graph-theory]] 참조.

### (f) Dijkstra 최단 경로
미방문 노드 중 거리가 가장 작은 노드를 매번 골라 확장.
[[../shortest-path/shortest-path]] 참조.

### (g) Interval Coloring (구간 색칠)
구간 N 개에 색을 칠해 겹치는 구간끼리 다른 색이 되게. 시작 시간 정렬 + 가장
빨리 끝나는 색 재사용.

### (h) 1이 될 때까지 / 만들 수 없는 금액 / 볼링공 고르기
이코테 Ch.11 의 그리디 변형 — 매 단계 가장 이득인 연산 선택.

---

## 9. 안티패턴 / 자주 틀리는 함정

### 함정 1 — 정당성 증명 없이 그리디 적용

```
잘못된 예: "동전 1, 3, 4 로 6원 거스름돈" 에 그리디 적용
→ 4 + 1 + 1 = 3 개 (잘못된 답)
정답: 3 + 3 = 2 개 (DP 가 맞음)
```

**대응**: 동전 단위가 배수 관계인지 먼저 확인. 작은 반례 (N=5 ~ 20)로 그리디
검증.

### 함정 2 — 정렬 기준이 틀림

회의실 배정에서 **시작 시간 정렬은 틀린다**:

```
회의: [1, 10] [2, 3] [3, 4]
시작 시간 정렬 → [1,10] 선택 → 끝났음 (1 개)
끝 시간 정렬 → [2,3] [3,4] 선택 (2 개)
```

### 함정 3 — 자료형 / 오버플로

큰 수 곱셈 / 누적 합에서 `int` 오버플로:
- Java/C/C++: `int` (≈ 21 억) → `long` / `long long` 사용
- Python 은 자동 big int (안전)
- Go: `int` 는 64bit on 64-bit OS, 명시는 `int64`
- Rust: `i32` → `i64` / `u64`

### 함정 4 — 부동소수점 비교

비율 / 평균 계산 시 `==` 비교는 위험.
- 정수 곱셈으로 변환: `a/b == c/d` → `a*d == c*b`
- 또는 `epsilon` 비교: `abs(x - y) < 1e-9`

### 함정 5 — 동전이 충분히 있나? 음수가 가능한가?

문제 조건 누락:
- 동전 개수 제한 → 무한이라 가정 X
- 음수 / 0 입력 → 별도 처리

### 함정 6 — 우선순위 큐 잘못 사용

최소 vs 최대 힙 — Python `heapq` 는 **최소 힙만**. 최대 힙은 `-value` 로
변환 또는 `heapq._heapify_max` 사용 (비공식).

---

## 10. 문제 유형 별 접근 절차 (실전 풀이 5 단계)

코테에서 그리디 문제를 만났을 때:

### Step 1 — 문제 분해
- **무엇을 최적화?** (최소/최대 = 무엇)
- **무엇을 선택?** (회의 / 동전 / 작업 / 노드)
- **제약 조건은?** (시간 / 무게 / 개수 / 순서)

### Step 2 — 정렬 기준 후보 나열
- 끝나는 시간 / 시작 시간 / 길이 / 가치 / 단위가치 / 빈도 / 비용...
- 보통 2-3 개 후보가 떠오른다.

### Step 3 — 작은 예시로 검증
- N=3-5 입력으로 손풀이.
- 각 정렬 기준에 대해 답 비교.
- 반례 (counter-example) 가 나오면 그 기준은 X.

### Step 4 — 정당성 1 줄로 표현
- "이 정렬 기준이 최적인 이유는 OOO" 를 한 줄로 적을 수 있어야.
- 못 적으면 그리디가 아닐 수 있다 → DP 시도.

### Step 5 — 구현 + 엣지 케이스
- 빈 입력 / N=1 / 동률 / 음수 / 최댓값 입력 확인.

---

## 11. 대표 문제 (이코테 Ch.03 + Ch.11)

### Q1. 큰 수의 법칙 (Ch.03)
배열의 수를 M 번 더하되 같은 인덱스는 K 번 연속 사용 가능. 합 최대화.

```python
n, m, k = map(int, input().split())
data = sorted(list(map(int, input().split())), reverse=True)
first, second = data[0], data[1]
# (k+1) 묶음 단위 = first×k + second×1
groups = m // (k + 1)
extra = m % (k + 1)
result = groups * (first * k + second) + extra * first
print(result)
```
**복잡도**: O(N log N) (정렬). **포인트**: 수식으로 직접 계산 (시뮬레이션 X).

### Q2. 숫자 카드 게임 (Ch.03)
N×M 카드. 각 행에서 가장 작은 수를 뽑은 뒤, 그 중 가장 큰 수 선택.

```python
n, m = map(int, input().split())
result = 0
for _ in range(n):
    row = list(map(int, input().split()))
    result = max(result, min(row))
print(result)
```
**복잡도**: O(N×M).

### Q3. 1이 될 때까지 (Ch.03)
N → 1 까지. 매 단계 (1) N -= 1 또는 (2) N //= K. 최소 횟수.

```python
n, k = map(int, input().split())
result = 0
while n > 1:
    if n % k == 0:
        n //= k
        result += 1
    else:
        # K 의 배수 될 때까지 -1 한 번에 처리
        target = (n // k) * k
        result += n - target
        n = target
print(result)
```
**복잡도**: O(log_k N) — N 을 매번 K 로 나눔.

### Q4. 모험가 길드 (Ch.11)
공포도 X 인 모험가는 X 명 이상 그룹에 속해야 함. 그룹 최대 개수.

```python
n = int(input())
data = sorted(list(map(int, input().split())))
result, count = 0, 0
for x in data:
    count += 1
    if count >= x:
        result += 1
        count = 0
print(result)
```
**포인트**: 공포도 작은 순 정렬 + 그룹이 가득 차면 즉시 마감.

### Q5. 만들 수 없는 금액 (Ch.11)
N 개 동전으로 만들 수 없는 최소 양의 정수.

```python
n = int(input())
data = sorted(list(map(int, input().split())))
target = 1
for coin in data:
    if coin > target:
        break
    target += coin
print(target)
```
**포인트**: 누적합이 항상 target 미만이면 target 을 만들 수 없음. **수학적**
사고 — `[1, k]` 범위를 만들 수 있을 때 다음 동전 c <= k+1 이면 `[1, k+c]`
까지 가능.

### Q6. 볼링공 고르기 (Ch.11)
무게 다른 볼링공 N 개에서 2 개 선택. 같은 무게는 안 됨.

```python
n, m = map(int, input().split())
weights = list(map(int, input().split()))
count = [0] * (m + 1)
for w in weights:
    count[w] += 1
result = 0
for i in range(1, m + 1):
    n -= count[i]   # i 무게는 위에서 처리. 남은 공 수
    result += count[i] * n
print(result)
```
**포인트**: 같은 무게끼리 빼고 곱셈 — O(N + M).

---

## 12. 연습 문제 (난이도별)

### 입문
- [백준 11047 동전 0](https://www.acmicpc.net/problem/11047) — 거스름돈 그리디
- [백준 1931 회의실 배정](https://www.acmicpc.net/problem/1931) — Activity Selection
- [백준 11399 ATM](https://www.acmicpc.net/problem/11399) — 정렬 + 누적합

### 중급
- [백준 1700 멀티탭 스케줄링](https://www.acmicpc.net/problem/1700) — 미래를 보는 그리디
- [백준 2138 전구와 스위치](https://www.acmicpc.net/problem/2138) — 결정 그리디
- [백준 1339 단어 수학](https://www.acmicpc.net/problem/1339) — 자릿수 가중치 정렬

### 고급
- [백준 2109 순회강연](https://www.acmicpc.net/problem/2109) — 마감일 + 우선순위 큐
- [백준 1202 보석 도둑](https://www.acmicpc.net/problem/1202) — 정렬 + 힙
- [LeetCode 134 Gas Station](https://leetcode.com/problems/gas-station/) — 누적합 그리디
- [LeetCode 763 Partition Labels](https://leetcode.com/problems/partition-labels/) — 구간 그리디

### 카카오 / 삼성 기출
- [Programmers — 단속카메라](https://programmers.co.kr/learn/courses/30/lessons/42884)
- [Programmers — 큰 수 만들기](https://programmers.co.kr/learn/courses/30/lessons/42883)
- [Programmers — 조이스틱](https://programmers.co.kr/learn/courses/30/lessons/42860)

---

## 13. 학습 자료

### 책
- 이것이 취업을 위한 코딩 테스트다 (나동빈) **Chapter 03, 11**
- Introduction to Algorithms (CLRS) **Chapter 16 — Greedy Algorithms**
- Algorithm Design (Kleinberg–Tardos) **Chapter 4**

### 강의
- [동빈나 — 그리디 강의 (무료)](https://www.youtube.com/watch?v=2zjoKjt97vQ)
- [Stanford CS161 — Greedy](https://stanford-cs161.github.io/winter2022/) — 학술
- [MIT 6.006 — Greedy](https://ocw.mit.edu/courses/6-006-introduction-to-algorithms-spring-2020/) — 영문

### 시각화
- [VisuAlgo — Greedy](https://visualgo.net/) — 한국어 지원
- [Algorithm Visualizer — Greedy](https://algorithm-visualizer.org/greedy)

### 깊이 학습
- [Greedy vs DP 비교 (GeeksforGeeks)](https://www.geeksforgeeks.org/greedy-vs-dynamic-programming-approach/)
- [Exchange Argument 증명 패턴](https://jeffe.cs.illinois.edu/teaching/algorithms/) — Jeff Erickson 무료 책

---

## 14. 관련

- [[../sorting/sorting]] — 그리디의 거의 모든 응용은 정렬 후 진행
- [[../dynamic-programming/dynamic-programming]] — 그리디가 안 될 때
- [[../shortest-path/shortest-path]] — Dijkstra 의 그리디 본질
- [[../graph-theory/graph-theory]] — Kruskal / Prim MST 그리디
- [[../implementation/implementation]] — 그리디 + 시뮬레이션 결합 문제
