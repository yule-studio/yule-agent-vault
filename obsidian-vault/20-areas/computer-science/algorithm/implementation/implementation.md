---
title: "구현 (Implementation) / 시뮬레이션"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T00:00:00+09:00
tags:
  - algorithm
  - implementation
  - simulation
---

# 구현 (Implementation) / 시뮬레이션

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.2.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 수준 재작성 + 11 언어 |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 최초 — 이코테 Ch.04/12 |

**[[../algorithm|↑ algorithm 인덱스]]**

---

## 1. 한 줄 정의

**알고리즘 자체는 단순한데, 그 알고리즘을 코드로 옮기는 것이 까다로운**
문제 유형. 보통 시뮬레이션 / 완전 탐색 / 격자 위 객체 이동 / 문자열
처리 / 좌표 회전이 등장한다.

---

## 2. 구현은 언제 쓰는가? — 시그널 인식

### (a) 문제 유형 키워드

- **"~을 그대로 옮겨라"** / "다음 규칙대로 시뮬레이션하라"
- **격자 / 보드 / 맵 위 객체 이동** ("LRUD" / "북동남서")
- **시계열 시뮬레이션** ("t = 0, 1, 2, ... 매 초마다")
- **카카오 1번 문제** (문자열 가공 / 진법 변환 / 압축 / 직사각형 회전)
- **삼성 SW 역량테스트** (대부분이 구현 + 완전 탐색)

### (b) 다른 알고리즘과 구분

| 차원 | 구현 | DFS/BFS | DP |
| --- | --- | --- | --- |
| 핵심 | 규칙대로 옮기기 | 그래프 탐색 | 부분 문제 |
| 길이 | 코드 길어짐 (50-150 줄) | 표준 패턴 (30-50 줄) | 점화식 + 표 |
| 사고 | 빠짐없이 케이스 분기 | 시작점부터 확장 | 작은 → 큰 |
| 신호 | "그대로 옮겨라" | "최단 거리 / 연결 요소" | "최적값 / 경우의 수" |

종종 구현 + DFS/BFS / 구현 + 그리디 결합 형태로 나온다 (카카오 1-2 번).

### (c) 카테고리

1. **격자 이동 시뮬레이션** — `dx/dy` + 경계 체크
2. **객체 상태 시뮬레이션** — 객체 N 개 (뱀, 좀비, 차) 의 시간 진화
3. **완전 탐색 (브루트포스)** — 가능한 모든 조합 / 순열 / 부분집합
4. **문자열 가공** — 진법 변환 / 압축 / 회전
5. **좌표 / 격자 회전** — 시계 90° / 대칭

---

## 3. 핵심 직관 — "사람이 손으로 풀듯이 컴퓨터에게"

구현은 "내가 종이 위에서 직접 풀이를 시뮬레이션할 수 있다면" → 그걸 그대로
코드로 옮긴다. 머리에서 그림이 그려지면 50% 푼 것.

비유:
- **요리 레시피** — 1단계 → 2단계 → ... 그대로.
- **보드게임** — 말 위치 / 점수 / 차례를 매번 갱신.

---

## 4. 핵심 패턴 5 가지 — 깊이 들여다보기

### 패턴 1: 방향 벡터 (`dx, dy`)

격자에서 상하좌우 이동은 4 개의 (행, 열) 변화량 배열로 표현.

```
       ↑ (-1, 0)
       │
(0,-1) ←┼→ (0, +1)
       │
       ↓ (+1, 0)
```

```python
# 상하좌우
dx = [-1, 1, 0, 0]
dy = [0, 0, -1, 1]

# 대각선 포함 8 방향
dx8 = [-1, -1, -1, 0, 0, 1, 1, 1]
dy8 = [-1, 0, 1, -1, 1, -1, 0, 1]

# 나이트 (체스) 8 방향
dx_knight = [-2, -2, -1, -1, 1, 1, 2, 2]
dy_knight = [-1, 1, -2, 2, -2, 2, -1, 1]
```

**경계 체크는 반드시**:
```python
if 0 <= nx < N and 0 <= ny < M:
    # 이동 가능
```

### 패턴 2: 객체 상태 + 시뮬레이션 루프

객체에 `(x, y, direction, time, energy, ...)` 등의 상태가 있다. 매 시간
단계마다 상태를 갱신.

```python
class Snake:
    def __init__(self, x, y, direction):
        self.body = deque([(x, y)])
        self.direction = direction  # 0=북, 1=동, 2=남, 3=서

def step(snake, board, turns, t):
    nx, ny = snake.body[0][0] + dx[snake.direction], snake.body[0][1] + dy[snake.direction]
    # 경계 / 자기 몸 / 사과 처리
    ...
```

### 패턴 3: 완전 탐색 / 순열·조합

작은 N (보통 N ≤ 10-15) 에서 모든 경우를 시도.

```python
from itertools import permutations, combinations

# 순열 (순서 O)
for p in permutations(range(n)):
    ...

# 조합 (순서 X)
for c in combinations(range(n), k):
    ...

# 부분집합 (비트마스킹)
for mask in range(1 << n):
    subset = [i for i in range(n) if mask & (1 << i)]
    ...
```

### 패턴 4: 격자 회전 / 대칭

90° 시계 방향 회전: `(r, c) → (c, N-1-r)`.
반시계: `(r, c) → (N-1-c, r)`.
좌우 대칭: `(r, c) → (r, N-1-c)`.

```python
def rotate_cw(matrix):
    n = len(matrix)
    return [[matrix[n-1-j][i] for j in range(n)] for i in range(n)]
```

### 패턴 5: 진법 변환 / 문자열 가공

```python
# 10진수 N → K진수 문자열
def to_base(n, k):
    digits = "0123456789ABCDEF"
    result = ""
    while n > 0:
        result = digits[n % k] + result
        n //= k
    return result or "0"
```

---

## 5. 복잡도 분석

| 유형 | 시간 | 공간 |
| --- | --- | --- |
| 격자 N×N 한 번 순회 | `O(N^2)` | `O(N^2)` (보드 저장) |
| 격자 + T 시간 시뮬레이션 | `O(N^2 × T)` | `O(N^2)` |
| 완전 탐색 — 순열 | `O(N!)` | `O(N)` |
| 완전 탐색 — 부분집합 | `O(2^N × N)` | `O(N)` |
| 격자 회전 1 회 | `O(N^2)` | `O(N^2)` |

**입력 크기로 후보 좁히기**:
- N ≤ 10 → 순열 / 백트래킹 OK.
- N ≤ 20 → 부분집합 비트마스킹 OK.
- N ≤ 500 → O(N^3) 시뮬레이션 가능.
- N ≤ 5000 → O(N^2 log N) 까지.

---

## 6. 의사 코드 (격자 시뮬레이션 표준)

```
function SIMULATE(grid, start, commands):
    pos ← start
    direction ← initial_direction
    for command in commands:
        if command == "이동":
            new_pos ← pos + dir_vector[direction]
            if new_pos 가 격자 안 and 장애물 아님:
                pos ← new_pos
        elif command == "회전":
            direction ← (direction + delta) mod 4
        elif command == "특수 동작":
            ...
    return pos
```

---

## 7. 언어별 구현 — "왕실의 나이트" + "캐릭터 이동" (11 언어)

체스판 8×8 에 나이트 위치 입력 → 갈 수 있는 위치 개수.

### Python 3
```python
def knight_moves(loc: str) -> int:
    col = ord(loc[0]) - ord('a') + 1   # a→1, b→2, ...
    row = int(loc[1])
    moves = [(-2,-1),(-2,1),(-1,-2),(-1,2),(1,-2),(1,2),(2,-1),(2,1)]
    return sum(1 for dr, dc in moves if 1 <= row+dr <= 8 and 1 <= col+dc <= 8)

# 캐릭터 이동
def character_move(n: int, plan: list[str]) -> tuple[int, int]:
    dx = {'L':0, 'R':0, 'U':-1, 'D':1}
    dy = {'L':-1, 'R':1, 'U':0, 'D':0}
    x, y = 1, 1
    for cmd in plan:
        nx, ny = x + dx[cmd], y + dy[cmd]
        if 1 <= nx <= n and 1 <= ny <= n:
            x, y = nx, ny
    return (x, y)

print(knight_moves('a1'))                              # 2
print(character_move(5, ['R','R','R','U','D','D']))    # (3, 4)
```

### Java
```java
import java.util.*;

public class Implementation {
    static int knightMoves(String loc) {
        int col = loc.charAt(0) - 'a' + 1;
        int row = loc.charAt(1) - '0';
        int[][] moves = {{-2,-1},{-2,1},{-1,-2},{-1,2},{1,-2},{1,2},{2,-1},{2,1}};
        int count = 0;
        for (int[] m : moves) {
            int nr = row + m[0], nc = col + m[1];
            if (1 <= nr && nr <= 8 && 1 <= nc && nc <= 8) count++;
        }
        return count;
    }

    static int[] characterMove(int n, String[] plan) {
        Map<String, int[]> d = Map.of(
            "L", new int[]{0,-1}, "R", new int[]{0,1},
            "U", new int[]{-1,0}, "D", new int[]{1,0}
        );
        int x = 1, y = 1;
        for (String c : plan) {
            int nx = x + d.get(c)[0], ny = y + d.get(c)[1];
            if (1 <= nx && nx <= n && 1 <= ny && ny <= n) { x = nx; y = ny; }
        }
        return new int[]{x, y};
    }

    public static void main(String[] args) {
        System.out.println(knightMoves("a1"));   // 2
        int[] r = characterMove(5, new String[]{"R","R","R","U","D","D"});
        System.out.println(r[0] + " " + r[1]);   // 3 4
    }
}
```

### C++
```cpp
#include <bits/stdc++.h>
using namespace std;

int knightMoves(string loc) {
    int col = loc[0] - 'a' + 1;
    int row = loc[1] - '0';
    int moves[8][2] = {{-2,-1},{-2,1},{-1,-2},{-1,2},{1,-2},{1,2},{2,-1},{2,1}};
    int count = 0;
    for (auto& m : moves) {
        int nr = row + m[0], nc = col + m[1];
        if (1 <= nr && nr <= 8 && 1 <= nc && nc <= 8) count++;
    }
    return count;
}

pair<int,int> characterMove(int n, vector<char> plan) {
    map<char, pair<int,int>> d = {
        {'L',{0,-1}},{'R',{0,1}},{'U',{-1,0}},{'D',{1,0}}
    };
    int x = 1, y = 1;
    for (char c : plan) {
        int nx = x + d[c].first, ny = y + d[c].second;
        if (1 <= nx && nx <= n && 1 <= ny && ny <= n) x = nx, y = ny;
    }
    return {x, y};
}

int main() {
    cout << knightMoves("a1") << endl;                                  // 2
    auto [x, y] = characterMove(5, {'R','R','R','U','D','D'});
    cout << x << " " << y << endl;                                       // 3 4
}
```

### C
```c
#include <stdio.h>
#include <string.h>

int knight_moves(const char *loc) {
    int col = loc[0] - 'a' + 1;
    int row = loc[1] - '0';
    int moves[8][2] = {{-2,-1},{-2,1},{-1,-2},{-1,2},{1,-2},{1,2},{2,-1},{2,1}};
    int count = 0;
    for (int i = 0; i < 8; i++) {
        int nr = row + moves[i][0], nc = col + moves[i][1];
        if (1 <= nr && nr <= 8 && 1 <= nc && nc <= 8) count++;
    }
    return count;
}

void character_move(int n, const char *plan, int *out_x, int *out_y) {
    int x = 1, y = 1;
    int dx[] = {0, 0, -1, 1};   // L R U D
    int dy[] = {-1, 1, 0, 0};
    const char *dir = "LRUD";
    for (int i = 0; plan[i]; i++) {
        const char *p = strchr(dir, plan[i]);
        if (!p) continue;
        int j = p - dir;
        int nx = x + dx[j], ny = y + dy[j];
        if (1 <= nx && nx <= n && 1 <= ny && ny <= n) { x = nx; y = ny; }
    }
    *out_x = x; *out_y = y;
}

int main() {
    printf("%d\n", knight_moves("a1"));                  // 2
    int x, y;
    character_move(5, "RRRUDD", &x, &y);
    printf("%d %d\n", x, y);                              // 3 4
}
```

### JavaScript (Node.js)
```javascript
function knightMoves(loc) {
    const col = loc.charCodeAt(0) - 'a'.charCodeAt(0) + 1;
    const row = parseInt(loc[1]);
    const moves = [[-2,-1],[-2,1],[-1,-2],[-1,2],[1,-2],[1,2],[2,-1],[2,1]];
    return moves.filter(([dr, dc]) =>
        1 <= row+dr && row+dr <= 8 && 1 <= col+dc && col+dc <= 8).length;
}

function characterMove(n, plan) {
    const d = {L:[0,-1], R:[0,1], U:[-1,0], D:[1,0]};
    let [x, y] = [1, 1];
    for (const c of plan) {
        const [dx, dy] = d[c];
        const [nx, ny] = [x + dx, y + dy];
        if (1 <= nx && nx <= n && 1 <= ny && ny <= n) [x, y] = [nx, ny];
    }
    return [x, y];
}

console.log(knightMoves('a1'));                                  // 2
console.log(characterMove(5, ['R','R','R','U','D','D']));        // [3, 4]
```

### TypeScript
```typescript
function knightMoves(loc: string): number {
    const col = loc.charCodeAt(0) - 'a'.charCodeAt(0) + 1;
    const row = parseInt(loc[1]);
    const moves: [number, number][] = [[-2,-1],[-2,1],[-1,-2],[-1,2],[1,-2],[1,2],[2,-1],[2,1]];
    return moves.filter(([dr, dc]) =>
        1 <= row+dr && row+dr <= 8 && 1 <= col+dc && col+dc <= 8).length;
}

function characterMove(n: number, plan: string[]): [number, number] {
    const d: Record<string, [number, number]> = {
        L: [0,-1], R: [0,1], U: [-1,0], D: [1,0]
    };
    let [x, y] = [1, 1];
    for (const c of plan) {
        const [dx, dy] = d[c];
        const [nx, ny] = [x + dx, y + dy];
        if (1 <= nx && nx <= n && 1 <= ny && ny <= n) [x, y] = [nx, ny];
    }
    return [x, y];
}

console.log(knightMoves('a1'));
console.log(characterMove(5, ['R','R','R','U','D','D']));
```

### Kotlin
```kotlin
fun knightMoves(loc: String): Int {
    val col = loc[0] - 'a' + 1
    val row = loc[1].digitToInt()
    val moves = listOf(-2 to -1, -2 to 1, -1 to -2, -1 to 2, 1 to -2, 1 to 2, 2 to -1, 2 to 1)
    return moves.count { (dr, dc) -> row + dr in 1..8 && col + dc in 1..8 }
}

fun characterMove(n: Int, plan: List<Char>): Pair<Int, Int> {
    val d = mapOf('L' to (0 to -1), 'R' to (0 to 1), 'U' to (-1 to 0), 'D' to (1 to 0))
    var (x, y) = 1 to 1
    for (c in plan) {
        val (dx, dy) = d[c] ?: continue
        val (nx, ny) = x + dx to y + dy
        if (nx in 1..n && ny in 1..n) { x = nx; y = ny }
    }
    return x to y
}

fun main() {
    println(knightMoves("a1"))                                            // 2
    println(characterMove(5, listOf('R','R','R','U','D','D')))             // (3, 4)
}
```

### Swift
```swift
func knightMoves(_ loc: String) -> Int {
    let chars = Array(loc)
    let col = Int(chars[0].asciiValue! - Character("a").asciiValue!) + 1
    let row = Int(String(chars[1]))!
    let moves: [(Int, Int)] = [(-2,-1),(-2,1),(-1,-2),(-1,2),(1,-2),(1,2),(2,-1),(2,1)]
    return moves.filter { (dr, dc) in
        let nr = row + dr, nc = col + dc
        return (1...8).contains(nr) && (1...8).contains(nc)
    }.count
}

func characterMove(_ n: Int, _ plan: [Character]) -> (Int, Int) {
    let d: [Character: (Int, Int)] = ["L":(0,-1), "R":(0,1), "U":(-1,0), "D":(1,0)]
    var (x, y) = (1, 1)
    for c in plan {
        guard let (dx, dy) = d[c] else { continue }
        let (nx, ny) = (x + dx, y + dy)
        if (1...n).contains(nx) && (1...n).contains(ny) { x = nx; y = ny }
    }
    return (x, y)
}

print(knightMoves("a1"))
print(characterMove(5, ["R","R","R","U","D","D"]))
```

### Go
```go
package main

import "fmt"

func knightMoves(loc string) int {
    col := int(loc[0] - 'a' + 1)
    row := int(loc[1] - '0')
    moves := [8][2]int{{-2,-1},{-2,1},{-1,-2},{-1,2},{1,-2},{1,2},{2,-1},{2,1}}
    count := 0
    for _, m := range moves {
        nr, nc := row+m[0], col+m[1]
        if 1 <= nr && nr <= 8 && 1 <= nc && nc <= 8 {
            count++
        }
    }
    return count
}

func characterMove(n int, plan []byte) (int, int) {
    d := map[byte][2]int{'L':{0,-1},'R':{0,1},'U':{-1,0},'D':{1,0}}
    x, y := 1, 1
    for _, c := range plan {
        nx, ny := x+d[c][0], y+d[c][1]
        if 1 <= nx && nx <= n && 1 <= ny && ny <= n {
            x, y = nx, ny
        }
    }
    return x, y
}

func main() {
    fmt.Println(knightMoves("a1"))                                  // 2
    x, y := characterMove(5, []byte("RRRUDD"))
    fmt.Println(x, y)                                                // 3 4
}
```

### Rust
```rust
fn knight_moves(loc: &str) -> u32 {
    let b = loc.as_bytes();
    let col = (b[0] - b'a' + 1) as i32;
    let row = (b[1] - b'0') as i32;
    let moves: [(i32, i32); 8] = [(-2,-1),(-2,1),(-1,-2),(-1,2),(1,-2),(1,2),(2,-1),(2,1)];
    moves.iter().filter(|&&(dr, dc)| {
        let (nr, nc) = (row + dr, col + dc);
        (1..=8).contains(&nr) && (1..=8).contains(&nc)
    }).count() as u32
}

fn character_move(n: i32, plan: &[char]) -> (i32, i32) {
    let mut x = 1; let mut y = 1;
    for &c in plan {
        let (dx, dy) = match c {
            'L' => (0, -1), 'R' => (0, 1),
            'U' => (-1, 0), 'D' => (1, 0),
            _ => continue,
        };
        let (nx, ny) = (x + dx, y + dy);
        if (1..=n).contains(&nx) && (1..=n).contains(&ny) { x = nx; y = ny; }
    }
    (x, y)
}

fn main() {
    println!("{}", knight_moves("a1"));                       // 2
    let r = character_move(5, &['R','R','R','U','D','D']);
    println!("{} {}", r.0, r.1);                              // 3 4
}
```

### C#
```csharp
using System;
using System.Linq;

public class Implementation {
    public static int KnightMoves(string loc) {
        int col = loc[0] - 'a' + 1;
        int row = loc[1] - '0';
        var moves = new (int, int)[] {(-2,-1),(-2,1),(-1,-2),(-1,2),(1,-2),(1,2),(2,-1),(2,1)};
        return moves.Count(m => 1 <= row+m.Item1 && row+m.Item1 <= 8
                              && 1 <= col+m.Item2 && col+m.Item2 <= 8);
    }

    public static (int, int) CharacterMove(int n, char[] plan) {
        var d = new System.Collections.Generic.Dictionary<char, (int, int)> {
            {'L',(0,-1)},{'R',(0,1)},{'U',(-1,0)},{'D',(1,0)}
        };
        int x = 1, y = 1;
        foreach (var c in plan) {
            var (dx, dy) = d[c];
            int nx = x + dx, ny = y + dy;
            if (1 <= nx && nx <= n && 1 <= ny && ny <= n) { x = nx; y = ny; }
        }
        return (x, y);
    }

    public static void Main() {
        Console.WriteLine(KnightMoves("a1"));                                       // 2
        var r = CharacterMove(5, new[] {'R','R','R','U','D','D'});
        Console.WriteLine($"{r.Item1} {r.Item2}");                                  // 3 4
    }
}
```

---

## 8. 변형 / 응용

### (a) 게임 개발 (이코테 Ch.04)
캐릭터가 방향 회전 + 전진 + 후진 (4 방향) 시뮬레이션. 모든 칸 가본 적
있는지 체크.

### (b) 뱀 (Ch.12, 삼성)
머리는 매 초 전진. 사과 먹으면 꼬리 안 줄임. 자기 몸 / 벽 부딪히면 종료.
**deque** 로 머리 / 꼬리 관리.

### (c) 자물쇠와 열쇠 (Ch.12, 카카오)
열쇠를 0°/90°/180°/270° 회전 + 격자 위 모든 위치에 시도. 자물쇠와 결합
시 모든 칸이 1 되면 OK.

### (d) 기둥과 보 설치 (Ch.12, 카카오)
설치 / 삭제 명령을 그대로 시뮬레이션. 설치 후 모든 구조물이 조건 만족하는지
검증.

### (e) 치킨 배달 (Ch.12, 삼성)
도시의 치킨집 중 M 개 선택 → 모든 집의 최단 치킨 거리 합 최소화. **조합
완전 탐색 + 거리 계산**.

### (f) 인구 이동 (Ch.13, 삼성)
연합 = BFS, 인구 평균화 = 시뮬레이션. **시뮬레이션 + BFS 결합**.

### (g) 블록 이동하기 (Ch.13, 카카오)
2 칸 짜리 로봇 + 회전. **상태가 (위치 + 방향)** 이라 BFS 의 상태 공간이
큼.

### (h) 진법 / 압축 / 회전 시리즈
- **럭키 스트레이트** (Ch.12) — 자릿수 합 비교
- **문자열 재정렬** (Ch.12) — 정렬 + 누적
- **문자열 압축** (Ch.12) — 모든 압축 단위 시도

---

## 9. 자주 틀리는 함정 / 안티패턴

### 함정 1 — 경계 체크 누락

```python
# 잘못
nx, ny = x + dx, y + dy
board[nx][ny] = 1   # IndexError 가능

# 옳음
nx, ny = x + dx, y + dy
if 0 <= nx < N and 0 <= ny < M:
    board[nx][ny] = 1
```

### 함정 2 — 0-based vs 1-based 인덱스

문제마다 "1부터 시작" 또는 "0부터 시작" 이 다르다. 입력 받자마자 통일.

```python
# 문제는 1-based 인데 배열은 0-based 인 경우
board = [[0] * (N+1) for _ in range(N+1)]   # 1..N 사용
```

### 함정 3 — 좌표 (x, y) vs (row, col)

- 수학: `(x, y)` = (가로, 세로)
- 격자: `(row, col)` = (세로, 가로)
- **혼동 주의** — 코드 안에서 `(r, c)` 또는 `(y, x)` 한 가지로 통일.

### 함정 4 — 방향 배열 순서가 일관되지 않음

```python
# 문제 1 에서 dx, dy 가 [북, 동, 남, 서] 였는데
# 문제 2 에서 [상, 하, 좌, 우] 로 다르게 정의하면 버그

# 항상 같은 순서 강제 (예: 북동남서 = 0, 1, 2, 3)
NORTH, EAST, SOUTH, WEST = 0, 1, 2, 3
dx = [-1, 0, 1, 0]
dy = [0, 1, 0, -1]
```

### 함정 5 — 시뮬레이션 종료 조건

- "벽에 부딪히면 종료" + "자기 몸에 부딪히면 종료" — 두 조건 모두 체크
- 종료 시점의 시간 (`t` 값) 출력에 +1 주의

### 함정 6 — `deque` 대신 `list` 의 `pop(0)`

```python
# 잘못 - O(N)
q = []
q.append(x)
front = q.pop(0)   # O(N) — 매우 느림

# 옳음 - O(1)
from collections import deque
q = deque()
q.append(x)
front = q.popleft()
```

### 함정 7 — 격자 회전 시 N x N 가정

직사각형 (N x M) 인데 정사각형 가정 → 회전 후 차원 바뀜.

```python
# 정사각형 가정
def rotate(matrix):
    n = len(matrix)
    return [[matrix[n-1-j][i] for j in range(n)] for i in range(n)]

# 직사각형 안전
def rotate_rect(matrix):
    rows, cols = len(matrix), len(matrix[0])
    return [[matrix[rows-1-j][i] for j in range(rows)] for i in range(cols)]
```

### 함정 8 — `visited` 배열 깊은/얕은 복사

```python
# 잘못 - 모든 행이 같은 리스트 가리킴
visited = [[False] * N] * N
visited[0][0] = True
# visited[1][0] 도 True 가 됨!

# 옳음
visited = [[False] * N for _ in range(N)]
```

---

## 10. 실전 풀이 절차 — 7 단계

### Step 1: 문제 정독 (2-3 회)
- 입력 / 출력 / 제한 / 예시 케이스를 종이에 적어둔다.
- "정확히 무엇을 시뮬레이션하라는 거지?" 한 문장으로 정리.

### Step 2: 데이터 구조 결정
- 격자 → 2D 배열
- 객체 N 개 → 클래스 / 튜플 리스트
- 큐 → `deque`
- 우선순위 → `heapq` / `PriorityQueue`

### Step 3: 손풀이 (예시 1-2 번)
- 입력 예시를 종이에 그려가며 풀어본다.
- "내 손으로 풀이 흐름" = 그대로 코드 흐름.

### Step 4: 헬퍼 함수 분리
- `is_valid(x, y)`, `rotate(d)`, `apply_command(c)` 등 잘게 쪼개기.
- main 루프가 깔끔해진다.

### Step 5: 작은 입력으로 디버그
- 예시 입력으로 실행 → 출력 비교.
- 안 맞으면 print 로 매 step 상태 출력.

### Step 6: 엣지 케이스
- 입력 N=1 / 빈 명령 / 경계 케이스 (격자 모서리)
- 큰 입력 (시간 초과 체크)

### Step 7: 코드 정리
- 변수명 명확히 / 매직 넘버 상수화 / 함수 분리.

---

## 11. 대표 문제 + 풀이 (이코테 Ch.04 + Ch.12)

### Q1. 상하좌우 (Ch.04)
N×N 격자, 시작 (1,1). LRUD 명령으로 이동.

```python
n = int(input())
plan = input().split()
x, y = 1, 1
dx = {'L':0, 'R':0, 'U':-1, 'D':1}
dy = {'L':-1, 'R':1, 'U':0, 'D':0}
for c in plan:
    nx, ny = x + dx[c], y + dy[c]
    if 1 <= nx <= n and 1 <= ny <= n:
        x, y = nx, ny
print(x, y)
```

### Q2. 시각 (Ch.04)
0 ~ N 시 사이 "3" 이 하나라도 들어간 모든 시각 카운트.

```python
n = int(input())
count = 0
for h in range(n + 1):
    for m in range(60):
        for s in range(60):
            if '3' in f"{h}{m}{s}":
                count += 1
print(count)
```
**복잡도**: O(N × 60 × 60) ≈ 86400 — 충분.

### Q3. 왕실의 나이트 (Ch.04)
위 11 언어 구현 참조.

### Q4. 게임 개발 (Ch.04)
4 방향 캐릭터가 좌회전 + 전진. 이미 가본 곳 or 바다면 후진.

```python
n, m = map(int, input().split())
x, y, d = map(int, input().split())
board = [list(map(int, input().split())) for _ in range(n)]
visited = [[0]*m for _ in range(n)]
visited[x][y] = 1
# 북0, 동1, 남2, 서3
dx = [-1, 0, 1, 0]
dy = [0, 1, 0, -1]

def turn_left():
    global d
    d = (d - 1) % 4

count = 1
turn_count = 0
while True:
    turn_left()
    nx, ny = x + dx[d], y + dy[d]
    if 0 <= nx < n and 0 <= ny < m and board[nx][ny] == 0 and visited[nx][ny] == 0:
        visited[nx][ny] = 1
        x, y = nx, ny
        count += 1
        turn_count = 0
    else:
        turn_count += 1
        if turn_count == 4:
            # 후진
            nx, ny = x - dx[d], y - dy[d]
            if 0 <= nx < n and 0 <= ny < m and board[nx][ny] == 0:
                x, y = nx, ny
                turn_count = 0
            else:
                break
print(count)
```

### Q5. 럭키 스트레이트 (Ch.12)
N 자리 수를 절반으로 나눠 자릿수 합 비교.

```python
n = input()
half = len(n) // 2
left = sum(map(int, n[:half]))
right = sum(map(int, n[half:]))
print("LUCKY" if left == right else "READY")
```

### Q6. 문자열 재정렬 (Ch.12)
문자열에서 알파벳 정렬 후 + 숫자 합 출력.

```python
s = input()
letters = []
digit_sum = 0
for c in s:
    if c.isalpha():
        letters.append(c)
    else:
        digit_sum += int(c)
letters.sort()
print(''.join(letters) + (str(digit_sum) if digit_sum > 0 else ''))
```

### Q7. 문자열 압축 (Ch.12, 카카오)
연속 같은 문자열 단위로 압축 — 모든 단위 시도해 최단 결과.

```python
def solution(s):
    answer = len(s)
    for step in range(1, len(s) // 2 + 1):
        compressed = ""
        prev = s[0:step]
        count = 1
        for j in range(step, len(s), step):
            if prev == s[j:j+step]:
                count += 1
            else:
                compressed += (str(count) if count > 1 else '') + prev
                prev = s[j:j+step]
                count = 1
        compressed += (str(count) if count > 1 else '') + prev
        answer = min(answer, len(compressed))
    return answer
```
**복잡도**: O(N^2) (각 step 마다 N 패스).

---

## 12. 연습 문제 (난이도별)

### 입문
- [백준 2738 행렬 덧셈](https://www.acmicpc.net/problem/2738)
- [백준 10818 최소 최대](https://www.acmicpc.net/problem/10818)
- [백준 2477 참외밭](https://www.acmicpc.net/problem/2477)

### 중급
- [백준 14503 로봇 청소기](https://www.acmicpc.net/problem/14503) — 4 방향 시뮬레이션
- [백준 14499 주사위 굴리기](https://www.acmicpc.net/problem/14499) — 주사위 상태 관리
- [백준 14890 경사로](https://www.acmicpc.net/problem/14890) — 격자 + 조건 체크
- [Programmers — 카카오 캐시](https://programmers.co.kr/learn/courses/30/lessons/17680)

### 고급 (삼성)
- [백준 16236 아기 상어](https://www.acmicpc.net/problem/16236) — BFS + 시뮬레이션
- [백준 19236 청소년 상어](https://www.acmicpc.net/problem/19236) — 백트래킹 + 시뮬레이션
- [백준 19237 어른 상어](https://www.acmicpc.net/problem/19237) — 격자 객체 N 개 시뮬레이션
- [백준 17143 낚시왕](https://www.acmicpc.net/problem/17143) — 복잡 시뮬레이션

### 카카오 기출
- [Programmers — 자물쇠와 열쇠](https://programmers.co.kr/learn/courses/30/lessons/60059)
- [Programmers — 기둥과 보 설치](https://programmers.co.kr/learn/courses/30/lessons/60061)
- [Programmers — 블록 이동하기](https://programmers.co.kr/learn/courses/30/lessons/60063)

---

## 13. 학습 자료

### 책 / 강의
- 이것이 취업을 위한 코딩 테스트다 (나동빈) **Ch.04, 12**
- [동빈나 — 구현 영상](https://www.youtube.com/watch?v=PnG6N0KbS5w) (무료)
- [백준 시뮬레이션 단계](https://www.acmicpc.net/step/2)

### 패턴 / 치트
- [Algorithm Cheatsheet — 격자 dx/dy](https://github.com/labuladong/fucking-algorithm) (중영)
- [구현 안티패턴 정리 (블로그)](https://wikidocs.net/170)

### 시각화
- [Snake 게임 시각화](https://playsnake.org/) — 뱀 문제 직관
- [Tetris 시뮬레이션](https://tetris.com/play-tetris) — 회전 패턴 직관

---

## 14. 관련

- [[../dfs-bfs/dfs-bfs]] — 격자 탐색 패턴 공유 (dx/dy 동일)
- [[../greedy/greedy]] — 시뮬레이션 + 그리디 결합 문제 다수
- [[../algorithm|↑ algorithm 인덱스]]
