---
title: "코딩 테스트 — 개념 / 배경 / 실습 환경"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T00:00:00+09:00
tags:
  - algorithm
  - coding-test
  - intro
---

# 코딩 테스트 — 개념 / 배경 / 실습 환경

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 최초 — 이코테 PART 01 Chapter 01 매핑 |

**[[../algorithm|↑ algorithm 인덱스]]**

## 1. 코딩 테스트 개념

**코딩 테스트** = 알고리즘 / 자료 구조 문제를 시간 제약 안에서 푸는 시험.
대부분의 IT 기업이 서류 다음 1차 관문으로 도입.

- **알고리즘 문제 풀이 (Problem Solving, PS)** — 주어진 문제를 코드로 해결.
- **시간 제한 (TLE)** — 보통 1-5 초. 입력 크기에 따라 정해진다.
- **메모리 제한 (MLE)** — 보통 128-512 MB.

## 2. 왜 도입되는가

- 단순 학력 / 자기소개서로 가려지지 않는 **문제 해결 능력** 측정.
- 채용 공정성 / 정량 평가 (점수화).
- 신입 채용 풀이 워낙 커서 자동 채점이 효율.

## 3. 출제 경향

| 영역 | 출제 빈도 | 이코테 챕터 |
| --- | --- | --- |
| 구현 / 시뮬레이션 | ⭐⭐⭐ | Ch.04, 12 |
| 그리디 | ⭐⭐⭐ | Ch.03, 11 |
| DFS / BFS | ⭐⭐⭐ | Ch.05, 13 |
| 다이나믹 프로그래밍 | ⭐⭐⭐ | Ch.08, 16 |
| 이진 탐색 | ⭐⭐ | Ch.07, 15 |
| 정렬 | ⭐⭐ | Ch.06, 14 |
| 최단 경로 | ⭐⭐ | Ch.09, 17 |
| 그래프 이론 | ⭐⭐ | Ch.10, 18 |

## 4. 실습 환경 구축

### Python (이코테 기본)
```bash
python3 --version          # 3.10+ 권장
pip install rich           # 디버깅 출력
```

### Java (대기업 백엔드)
```bash
sdk install java 21.0.5-tem   # SDKMAN
javac Main.java && java Main
```

### C++ (성능 우선 / 알고리즘 대회)
```bash
brew install gcc            # macOS
g++ -O2 -o main main.cpp && ./main
```

### C (임베디드 / 학교)
```bash
gcc -O2 -o main main.c && ./main
```

### 온라인 IDE
- [LeetCode Playground](https://leetcode.com/playground/)
- [Replit](https://replit.com/)
- [Programmers IDE](https://programmers.co.kr/) (한국)
- [Codeforces IDE](https://codeforces.com/)

## 5. 입출력 표준 (4 언어 비교)

### Python
```python
import sys
input = sys.stdin.readline   # 빠른 입력
n = int(input())
a = list(map(int, input().split()))
print(*a)
```

### Java
```java
import java.io.*;
import java.util.*;
public class Main {
    public static void main(String[] args) throws IOException {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        int n = Integer.parseInt(br.readLine());
        StringTokenizer st = new StringTokenizer(br.readLine());
        int[] a = new int[n];
        for (int i = 0; i < n; i++) a[i] = Integer.parseInt(st.nextToken());
        StringBuilder sb = new StringBuilder();
        for (int x : a) sb.append(x).append(' ');
        System.out.println(sb);
    }
}
```

### C++
```cpp
#include <bits/stdc++.h>
using namespace std;
int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(NULL);
    int n; cin >> n;
    vector<int> a(n);
    for (int i = 0; i < n; i++) cin >> a[i];
    for (int x : a) cout << x << ' ';
    return 0;
}
```

### C
```c
#include <stdio.h>
#include <stdlib.h>
int main() {
    int n; scanf("%d", &n);
    int *a = malloc(n * sizeof(int));
    for (int i = 0; i < n; i++) scanf("%d", &a[i]);
    for (int i = 0; i < n; i++) printf("%d ", a[i]);
    free(a);
    return 0;
}
```

## 6. 준비 자세

1. **문제 유형 분류 능력** — 처음 5 분 안에 "이건 DP" 또는 "이건 그리디"
   감지 가능해야.
2. **시간 복잡도 추정** — 입력 N 크기로 허용 알고리즘 후보를 좁힌다
   (N=10^5 → O(N log N) 이하 필요).
3. **표준 라이브러리 익숙** — Python `heapq` / Java `PriorityQueue` /
   C++ `std::priority_queue` / C 는 직접 구현.
4. **꾸준한 풀이** — 1 일 1-2 문제, 3 개월+ 누적.

## 7. 학습 자료

- [이것이 취업을 위한 코딩 테스트다 (책)](http://www.yes24.com/Product/Goods/91433923) — 한국 코테 표준 책
- [동빈나 YouTube](https://www.youtube.com/@dongbinna) — 책 저자, 한국어 강의
- [Programmers](https://programmers.co.kr/) — 한국 기업 기출 풀이 사이트
- [백준 BOJ](https://www.acmicpc.net/) — 알고리즘 문제 사이트 (한국 표준)
- [LeetCode](https://leetcode.com/) — 글로벌 표준 (해외 회사 대비)
- [Codeforces](https://codeforces.com/) — 알고리즘 대회

## 관련

- [[complexity]] — 시간 / 공간 복잡도
- [[interview-process]] — 기술 면접
- [[problem-solving-sites]] — 문제 풀이 사이트
