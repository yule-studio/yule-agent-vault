---
title: "복잡도 — 시간 / 공간 / Big-O"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T00:00:00+09:00
tags:
  - algorithm
  - complexity
  - big-o
  - intro
---

# 복잡도 — 시간 / 공간 / Big-O

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 최초 — 이코테 PART 01 Chapter 01-3 매핑 |

**[[../algorithm|↑ algorithm 인덱스]]**

## 1. 개요

복잡도 (complexity) = 알고리즘이 **입력 크기 N 에 따라 얼마나 시간을 쓰고
얼마나 메모리를 쓰는지** 의 측정. 코딩 테스트의 "TLE / MLE" 의 기준.

## 2. 점근 표기법

| 표기 | 의미 | 의도 |
| --- | --- | --- |
| **Big-O** `O(f(N))` | 상한 (worst case) | 가장 오래 걸리는 경우 |
| **Theta** `Θ(f(N))` | 평균 (tight bound) | 보통 경우 |
| **Omega** `Ω(f(N))` | 하한 (best case) | 가장 빠를 때 |

코딩 테스트는 거의 **Big-O 만** 사용 (worst case 기준).

## 3. 시간 복잡도 — 입력 크기 N 별 한도

| N 크기 | 허용 복잡도 | 알고리즘 후보 |
| --- | --- | --- |
| N ≤ 11 | O(N!) | 순열 / 백트래킹 완전탐색 |
| N ≤ 20 | O(2^N) | 비트마스킹 / 부분집합 |
| N ≤ 500 | O(N^3) | 플로이드-워셜 |
| N ≤ 2,000 | O(N^2 log N) | DP 일부 |
| N ≤ 10,000 | O(N^2) | 다이나믹 / 이중 반복 |
| N ≤ 100,000 | O(N log N) | 정렬 / 우선순위 큐 |
| N ≤ 10,000,000 | O(N) | 단일 순회 |
| N ≤ 10^18 | O(log N) | 이진 탐색 / 거듭제곱 |

**대략적 기준**: 1 초 안에 약 1 억 (10^8) 연산. Python 은 그 1/10 (10^7).

## 4. 자주 만나는 복잡도

```
O(1)       상수      — 배열 인덱스 접근
O(log N)   로그      — 이진 탐색
O(N)       선형      — 단일 for 루프
O(N log N) 선형 로그 — 효율적 정렬 (Merge / Quick / Heap)
O(N^2)     이차      — 이중 루프
O(N^3)     삼차      — 플로이드-워셜
O(2^N)     지수      — 부분집합
O(N!)      팩토리얼  — 순열
```

## 5. 공간 복잡도

- 보통 N ≤ 10^7 정수 배열 → 약 40 MB (int 4 byte 가정)
- DP 표 2 차원 N × N → N ≤ 5000 정도가 안전
- 메모리 제한 256 MB → int 약 6 × 10^7 개

## 6. 4 언어 — 시간 측정

### Python
```python
import time
start = time.time()
# ...
print(f"{time.time() - start:.3f}s")
```

### Java
```java
long start = System.currentTimeMillis();
// ...
System.out.println((System.currentTimeMillis() - start) + "ms");
```

### C++
```cpp
#include <chrono>
auto start = std::chrono::high_resolution_clock::now();
// ...
auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(
    std::chrono::high_resolution_clock::now() - start);
std::cout << elapsed.count() << "ms\n";
```

### C
```c
#include <time.h>
clock_t start = clock();
// ...
printf("%.3fs\n", (double)(clock() - start) / CLOCKS_PER_SEC);
```

## 7. 복잡도 계산 예제

### 예 1 — 이중 루프

```python
for i in range(n):
    for j in range(n):
        # O(1) 작업
```
→ `O(N^2)`. N=10000 이면 10^8 → Python 10 초 (TLE 가능성).

### 예 2 — 재귀

```python
def fib(n):
    if n <= 1: return n
    return fib(n-1) + fib(n-2)
```
→ `O(2^N)`. N=40 도 어렵다. → 메모이제이션 / DP 로 `O(N)`.

### 예 3 — 분할 정복 / 마스터 정리

```python
def merge_sort(a):
    if len(a) <= 1: return a
    mid = len(a) // 2
    left = merge_sort(a[:mid])     # T(N/2)
    right = merge_sort(a[mid:])    # T(N/2)
    return merge(left, right)      # O(N)
```
→ `T(N) = 2T(N/2) + O(N) = O(N log N)`.

## 8. 안티패턴 / 함정

- **Python list.append() 는 amortized O(1)** 이지만 `list.insert(0, x)` 는 O(N).
  큐가 필요하면 `collections.deque`.
- **String 연결**: Python `''.join(list)` vs `s += x` (후자는 O(N^2)).
  Java 도 `StringBuilder` 사용.
- **재귀 깊이**: Python 기본 1000. `sys.setrecursionlimit(10**6)`.
- **자료형 오버플로**: Java/C++ int 는 2^31 약 21억. `long` / `long long` 으로.
  Python 은 자동 big int.
- **메모리 초과**: N × N DP 표인데 N=10^5 면 10^10 셀 → 절대 불가.

## 9. 학습 자료

- [Big-O Cheat Sheet](https://www.bigocheatsheet.com/) — 자료구조 / 알고리즘 복잡도 표
- [Algorithm Visualizer](https://algorithm-visualizer.org/) — 시각화
- [VisuAlgo](https://visualgo.net/) — 알고리즘 시각화 (한국어 지원)
- [Introduction to Algorithms (CLRS)](https://mitpress.mit.edu/books/introduction-algorithms-fourth-edition) — 표준 책
- [이코테 책 Chapter 01-3](http://www.yes24.com/Product/Goods/91433923) — 한국어 개념 정리

## 관련

- [[coding-test-overview]] — 시간 제한과 N 크기 관계
- [[../sorting/sorting]] — 정렬의 N log N 의 직관
- [[../dynamic-programming/dynamic-programming]] — 지수 → 다항식 변환
