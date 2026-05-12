---
title: "배열 / 문자열 (Arrays & Strings) — 사전형"
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
| v.2.0.0 | 2026-05-13 | engineering-agent/tech-lead | **개념 / 이론 / 하드웨어 상호작용** 깊이 확장 |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 최초 — 11 언어 + 실습 |

**[[../data-structure|↑ data-structure]]** · **[[../../computer-science|↑↑ computer-science]]**

---

## 1. 한 줄 정의

**연속된 메모리 공간에 같은 타입의 값을 일정한 간격으로 나열**한 가장 기본
자료구조. 인덱스로 O(1) 접근. 모든 다른 자료구조의 토대이자 CPU 캐시 / SIMD /
GPU 메모리 모델이 직접 가정하는 자료 형태.

---

## 2. 개념의 깊이 — 왜 "배열" 이라는 추상이 존재하는가

### 2.1 컴퓨터 메모리의 본질

**컴퓨터 메모리는 본질적으로 거대한 1차원 배열이다.**

폰 노이만 구조 (1945) 의 기본 가정:
- 메모리 = 일렬로 나열된 **셀 (cell)** 의 모음.
- 각 셀은 **고유 주소 (address)** 를 가지고, 그 주소로 직접 접근 가능.
- "Random Access" — Random 은 "무작위" 가 아니라 **"임의의 주소를 직접 접근"**.

```
주소:    0x0000  0x0001  0x0002  ...  0xFFFE  0xFFFF
값:      [ 0xA3 ][ 0x00 ][ 0x4F ] ... [ 0x12 ][ 0xCD ]
         ↑ 1 byte 씩
```

배열은 이 메모리 모델의 **자연스러운 추상화** — "연속된 셀들에 같은 종류의
값을 담는 묶음".

### 2.2 추상 자료형 (Abstract Data Type, ADT) 이론

**Barbara Liskov & Stephen Zilles (1974)** 의 데이터 추상화 논문 이후, 자료구조는
두 층으로 분리:

| 층 | 정의 | 예시 |
| --- | --- | --- |
| **추상 자료형 (ADT)** | 인터페이스 — "무엇을 할 수 있나" | List, Set, Map, Stack |
| **자료구조 (Data Structure)** | 구현 — "어떻게 메모리에 표현하나" | Array, Linked List, Hash Table |

배열 ADT 의 인터페이스:
- `get(i)` — O(1) 무작위 접근
- `set(i, v)` — O(1) 갱신
- `length` — O(1) 크기 조회
- `iterate` — O(N) 순회

배열의 ADT 약속을 **메모리 연속 + 고정 크기 셀** 로 구현하면 위 인터페이스가
자연스럽게 O(1) 이 보장됨.

### 2.3 역사 — 배열의 탄생

- **1957 Fortran (John Backus)**: 최초의 고수준 언어. 행렬 계산을 위해 다차원
  배열을 1급 시민으로 도입. "Formula Translator" 의 핵심.
- **1958 ALGOL**: `array` 키워드 정식화.
- **1972 C (Dennis Ritchie)**: 배열을 **포인터의 syntactic sugar** 로 명시 —
  `a[i] == *(a + i)`. 메모리 모델과 1:1 대응.
- **1995 Java**: 배열을 객체로 — `length` 필드 + 경계 검사.
- **2000s 동적 배열의 대중화** — Python `list`, JavaScript `Array`, Java
  `ArrayList`, C++ `vector` 가 표준이 됨.

### 2.4 왜 "같은 타입" 인가?

각 원소가 **동일한 크기 (byte 수)** 여야 인덱스 → 주소 변환이 단일 곱셈으로
가능:

```
주소(a[i]) = base_address + i × sizeof(element)
```

만약 원소 크기가 다르면 (예: 가변 길이 객체):
- 각 원소의 위치를 알기 위해 **앞에서부터 누적 합** 필요 → O(N) 접근.
- 이런 경우는 **간접 접근** (포인터 배열) 으로 해결. Python `list` 가
  실제로는 PyObject 포인터의 배열.

### 2.5 배열 vs 다른 ADT 의 본질적 차이

| ADT | 본질 | 메모리 |
| --- | --- | --- |
| Array | "위치 (position) 가 의미" | 연속 |
| Linked List | "다음 (next) 이 의미" | 분산 |
| Tree | "부모-자식 관계가 의미" | 분산 |
| Hash Table | "키 → 위치 매핑" | 부분 연속 (bucket) |
| Graph | "임의 노드 간 관계" | 분산 |

배열은 **"순서가 의미"** 인 데이터에 최적 — 일렬 시계열 / 행렬 / 픽셀 / 토큰
시퀀스 등.

---

## 3. 이론적 배경 — 메모리 / CPU / 캐시 의 상호작용

### 3.1 메모리 계층 (Memory Hierarchy)

현대 CPU 는 **속도 vs 용량** 의 트레이드오프로 메모리를 계층화:

| 계층 | 크기 | 접근 시간 | 비용 |
| --- | --- | --- | --- |
| **레지스터** | ~1 KB | <1 ns | 매우 비쌈 |
| **L1 캐시** | 32-64 KB | 1-2 ns | 비쌈 |
| **L2 캐시** | 256 KB - 1 MB | 3-10 ns | |
| **L3 캐시** | 4-64 MB | 10-30 ns | |
| **RAM** | 8-128 GB | 50-100 ns | 보통 |
| **SSD** | 0.5-4 TB | 100,000 ns (=0.1 ms) | 저렴 |
| **HDD** | 1-20 TB | 10,000,000 ns (=10 ms) | 매우 저렴 |

> RAM 접근은 L1 보다 **100배 느리다**. 캐시 미스 1번 = 명령어 100개 실행 비용.

### 3.2 캐시 라인 (Cache Line)

CPU 는 메모리를 **64 byte 단위 (캐시 라인)** 로 가져온다. 한 byte 만 필요해도
64 byte 가 함께 로드됨.

```
배열 a = [int, int, int, int, ...]  (각 4 byte)

a[0] 접근 시:
  → CPU 가 a[0..15] (64 byte / 4 byte = 16개) 전체를 한 번에 캐시로 로드
  → 다음 a[1], a[2], ..., a[15] 접근은 캐시에서 직접 (L1 1ns)
```

이게 배열이 빠른 **근본 이유**.

### 3.3 공간 지역성 (Spatial Locality)

"이번에 접근한 주소 근처를 또 접근할 가능성이 높다" 는 경험적 사실 →
CPU 가 미리 캐시에 올려둠 (prefetching).

**배열 = 공간 지역성의 끝판왕** — 정확히 인접 주소만 접근.

연결 리스트 (Linked List) 와 비교:

```
배열:     [10]→[20]→[30]→[40]    ← 64 byte 안에 다 들어옴 (1 캐시 라인)
연결 리스트: [10|*]   [20|*]   [30|*]   [40|*]
             0x100    0x800    0x300    0x500   ← 흩어진 주소

배열 순회: 1 캐시 미스 + 15 캐시 히트 = 매우 빠름
리스트 순회: 매 노드마다 캐시 미스 = 매우 느림
```

실측: 같은 N=10^6 원소 순회 시 **배열이 리스트보다 10-100배 빠름**.

### 3.4 분기 예측 (Branch Prediction)

CPU 는 `if` 문의 다음 명령어를 미리 추측해 파이프라인에 넣는다 (speculative
execution). **정렬된 배열이 비정렬보다 빠른 유명한 사례** (Stack Overflow
인기 답변):

```java
// Mystery: 정렬된 배열이 5배 빠르다
int[] data = new int[32768];
for (int i = 0; i < data.length; i++) data[i] = rand.nextInt() % 256;
// Arrays.sort(data);   // 이 줄을 켜면 5배 빨라짐!

long sum = 0;
for (int i = 0; i < 100000; i++) {
    for (int j = 0; j < data.length; j++) {
        if (data[j] >= 128) sum += data[j];
    }
}
```

**이유**: 정렬되면 `data[j] >= 128` 의 결과가 같은 줄로 N/2 번 연속 → 분기 예측
적중률 ~100%. 비정렬이면 ~50% → 매번 파이프라인 flush.

### 3.5 SIMD / 벡터화

현대 CPU 의 **SIMD** (Single Instruction Multiple Data) — AVX2 (256-bit) / AVX-512
(512-bit) — 는 **연속된 메모리** 에서 4-16 개 값을 한 번에 연산.

```c
// 스칼라
for (int i = 0; i < n; i++) c[i] = a[i] + b[i];

// SIMD (AVX2 256-bit, 8개 int 동시)
for (int i = 0; i < n; i += 8) {
    __m256i va = _mm256_loadu_si256((__m256i*)&a[i]);
    __m256i vb = _mm256_loadu_si256((__m256i*)&b[i]);
    __m256i vc = _mm256_add_epi32(va, vb);
    _mm256_storeu_si256((__m256i*)&c[i], vc);
}
// 약 8배 빠름. 연결 리스트로는 절대 불가.
```

배열이 GPU 의 자료구조이기도 한 이유 — GPU 는 수천 개 스레드가 **연속
메모리 (coalesced access)** 를 가정.

### 3.6 가상 메모리 / 페이지

OS 는 메모리를 **페이지 (보통 4 KB)** 단위로 관리. 큰 배열이 페이지 경계를
넘으면 **TLB miss + page fault** 가능.

`mmap` 으로 거대 배열을 매핑하면 실제 메모리에 로드되기 전까지 가상 주소만
존재 — 디스크 파일을 배열처럼 다룰 수 있음 (NumPy `memmap`, Boost.Interprocess).

---

## 4. 정적 vs 동적 배열 — 설계 결정의 깊이

### 4.1 정적 배열 (Static Array)

```c
int a[100];   // 컴파일 타임에 크기 결정
```

- **장점**: 스택에 할당 가능 (할당/해제 비용 0), 단순
- **단점**: 크기 변경 불가. 큰 배열 (수 MB+) 은 스택 오버플로
- **사용처**: 임베디드 / 커널 / 작은 고정 크기 데이터 / SIMD

### 4.2 동적 배열 (Dynamic Array / Growable Array)

```cpp
std::vector<int> v;
v.push_back(10);   // 필요시 자동 확장
```

내부 동작:
1. 초기 capacity 4 (구현마다 다름).
2. size = capacity 가 되면 **resize**:
   - 새 메모리 할당 (capacity × growth_factor, 보통 1.5 or 2).
   - 기존 원소 N 개 복사 — O(N).
   - 옛 메모리 해제.
3. 새 capacity 까지는 O(1) push 가능.

### 4.3 Amortized Analysis — 수학적 증명

"왜 push_back 이 amortized O(1) 인가?"

**Aggregate Method (집합 방법)**:

N 번의 push 의 총 비용을 세고 N 으로 나눠 평균을 구한다.

```
capacity 변화: 1 → 2 → 4 → 8 → ... → 2^k (≥ N)

resize 비용 (복사):
  1 → 2: 1 개 복사
  2 → 4: 2 개 복사
  4 → 8: 4 개 복사
  ...
  2^(k-1) → 2^k: 2^(k-1) 개 복사

총 복사 비용 = 1 + 2 + 4 + ... + 2^(k-1) = 2^k - 1 < 2N

각 push 의 자체 비용 = O(1) (배열에 값 쓰기)

총 비용 = N × O(1) + 2N = O(N)
평균 = O(N) / N = O(1)
```

→ **각 push 는 평균 O(1)** (단, 개별 push 의 최악은 O(N) — resize 시).

**왜 2 배인가?**

- 1.5 배: 메모리 사용량 최소 (이전 메모리 재사용 가능 — Facebook folly 채택)
- 2 배: 캐시 / 메모리 할당자 친화 (대부분 표준 라이브러리)
- 4 배: resize 빈도 최소화이지만 메모리 낭비 큼

선택은 **수학적으로는 어느 상수 ≥ 1.5 이상이면 amortized O(1)** 보장. 1 배
증가 (예: `n + 100`) 는 amortized O(N) 으로 망가짐:

```
n + 100 씩 증가 시 총 복사 = 100 + 200 + 300 + ... + N ≈ N²/200 = O(N²)
→ 각 push 가 O(N) 평균 ← 매우 나쁨
```

### 4.4 Banker's Method (계좌 방법)

각 push 에 **3 단위 비용**을 미리 부과:
- 1 단위: 실제 push 비용.
- 2 단위: "예금" — resize 시 사용할 비축.

resize 가 일어나면 N 단위 비용이 발생하지만, 이전 N 번의 push 에서 2N 단위가
예금되어 있어 충분히 지불 가능. 평균 3 단위 = **amortized O(1)**.

---

## 5. 문자열 — 단순한 char 배열이 아니다

### 5.1 인코딩의 역사

| 연도 | 표준 | 크기 | 비고 |
| --- | --- | --- | --- |
| 1963 | **ASCII** | 7 bit (128 글자) | 영문 + 제어 문자만 |
| 1980s | EUC-KR / Shift-JIS | 2 byte 가변 | 동아시아 |
| 1991 | **Unicode** | 코드 포인트 (논리 번호) | 모든 문자 통합 |
| 1992 | **UTF-8** (Pike & Thompson) | 1-4 byte 가변 | ASCII 호환 |
| 1996 | **UTF-16** | 2-4 byte 가변 | Java / Windows / JS 내부 |
| 2000s | **UTF-32** | 4 byte 고정 | 인덱싱 O(1) but 메모리 4배 |

### 5.2 UTF-8 인코딩 — 천재적 설계

```
코드 포인트 범위         UTF-8 byte 패턴
U+0000  - U+007F       0xxxxxxx                  (ASCII 호환)
U+0080  - U+07FF       110xxxxx 10xxxxxx
U+0800  - U+FFFF       1110xxxx 10xxxxxx 10xxxxxx     (한글 여기)
U+10000 - U+10FFFF     11110xxx 10xxxxxx 10xxxxxx 10xxxxxx (이모지)
```

**특성**:
- 각 byte 의 시작 패턴만 봐도 **연속 byte 인지 / 첫 byte 인지** 즉시 판별.
- 어느 위치에서 잘려도 다음 첫 byte 까지 건너뛰면 복구 가능.
- ASCII 는 그대로 보존 — 기존 시스템 호환.

### 5.3 그래미 클러스터 (Grapheme Cluster)

사람이 "한 글자" 로 인식하는 단위 ≠ 코드 포인트 1개:

```
"한"   1 코드 포인트 (U+D55C) = 1 글자
"가" (조합형) = ㄱ(U+1100) + ㅏ(U+1161) = 2 코드 포인트 1 글자
"👨‍👩‍👧‍👦" (가족 이모지) = 7 코드 포인트 1 글자 (ZWJ 결합)
"é" = "e" + 결합 액센트 = 2 코드 포인트 1 글자
```

**`len("👨‍👩‍👧‍👦")` 의 결과는 언어마다 다름**:

| 언어 | 결과 | 의미 |
| --- | --- | --- |
| Python 3 | 7 | 코드 포인트 수 |
| Java `length()` | 11 | UTF-16 code unit 수 |
| Swift `count` | 1 | **Grapheme cluster** (제일 정확) |
| Rust `chars().count()` | 7 | 코드 포인트 |
| Go `len()` | 25 | UTF-8 byte 수 |

→ 사용자 문자 수가 중요하면 **Swift 또는 ICU 라이브러리** 사용.

### 5.4 immutable vs mutable 의 디자인 결정

| 언어 | 기본 | 이유 |
| --- | --- | --- |
| Python `str` | immutable | 해싱 가능 (dict 키), thread safety, **value semantics** |
| Java `String` | immutable | 위와 같음 + **String pool** (intern) |
| JavaScript `string` | immutable | 위와 같음 |
| Rust `String` | mutable | 명시적 mutation. 불변 원하면 `&str` |
| C++ `std::string` | mutable | 성능 우선 |
| C `char[]` | mutable | 저수준 |

**immutable 문자열 + interning** 의 효과:
```python
a = "hello"
b = "hello"
a is b   # True (같은 객체 — Python 이 작은 문자열 자동 intern)
```

**immutable 의 비용**: concat 시 매번 새 객체 → O(N+M). 누적 시 O(N²). 빌더
패턴 (StringBuilder / list+join) 사용.

### 5.5 짧은 문자열 최적화 (Small String Optimization, SSO)

대부분의 문자열은 짧다 (이름, 키워드 등). C++ `std::string` 은 보통 16 byte
이하 문자열을 **힙 할당 없이 객체 안에 직접 저장**:

```cpp
class string {
    union {
        char short_buf[16];     // 짧은 문자열 (15 글자 + \0)
        struct { char* ptr; size_t cap; } long_data;  // 긴 문자열
    };
    size_t size;
    bool is_short;
};
```

→ 짧은 문자열의 할당 비용 거의 0. 캐시 친화적.

같은 최적화: Rust `String` (X), Java compact string (Java 9+, ISO-Latin-1 압축).

---

## 6. 핵심 직관 — 메모리 레이아웃 + 다차원 배열

### 6.1 1D 배열 메모리

```
int a[5] = {10, 20, 30, 40, 50};

주소:    0x100  0x104  0x108  0x10C  0x110
값:      [10]   [20]   [30]   [40]   [50]

a[i] 접근:
  → 어셈블리 한 줄: mov eax, [base + i * 4]
  → CPU 한 사이클
```

### 6.2 2D 배열 — Row-major vs Column-major

```
m[3][4] = {
    {a, b, c, d},
    {e, f, g, h},
    {i, j, k, l}
}

Row-major (C, C++, Python NumPy 기본, Java):
  메모리: a b c d e f g h i j k l
  m[i][j] 주소 = base + (i × COLS + j) × sizeof(elem)
  → 같은 행 원소가 인접
  
Column-major (Fortran, MATLAB, R, OpenGL):
  메모리: a e i b f j c g k d h l
  m[i][j] 주소 = base + (j × ROWS + i) × sizeof(elem)
  → 같은 열 원소가 인접
```

**Loop 순서가 성능에 미치는 영향**:

```c
// C는 row-major. 이 순서가 캐시 친화적
for (i = 0; i < N; i++)
    for (j = 0; j < N; j++)
        sum += m[i][j];   // 빠름 (캐시 hit)

// 반대 순서
for (j = 0; j < N; j++)
    for (i = 0; i < N; i++)
        sum += m[i][j];   // 매우 느림 (매번 캐시 miss)
```

N=10000 행렬에서 두 순서의 성능 차이가 **10배 이상**.

### 6.3 행렬 곱셈 최적화 — Loop Reordering

```c
// 표준 (ikj 순서가 cache 친화적인 경우도 있음)
for (i = 0; i < N; i++)
    for (j = 0; j < N; j++)
        for (k = 0; k < N; k++)
            c[i][j] += a[i][k] * b[k][j];

// Cache blocking — 캐시에 맞는 작은 블록 단위로
for (i = 0; i < N; i += B)
    for (j = 0; j < N; j += B)
        for (k = 0; k < N; k += B)
            for (ii = i; ii < i+B; ii++)
                for (jj = j; jj < j+B; jj++)
                    for (kk = k; kk < k+B; kk++)
                        c[ii][jj] += a[ii][kk] * b[kk][jj];
```

이런 최적화로 BLAS 라이브러리 (OpenBLAS, MKL) 가 naive 구현보다 **수십~수백
배** 빠르다.

---

## 7. 동작 원리 — 자세한 예시

### 7.1 정적 배열

```c
int a[5];                # 메모리 5 슬롯 할당, 인접 — 20 byte
a[0] = 10;               # 0x100 위치에 10 저장 — O(1)
int x = a[2];            # 0x108 위치 읽기 — O(1)
a[3] = a[3] + 1;         # read 0x10C → +1 → write 0x10C — O(1)
크기 증가? → ❌ 불가 (a 는 컴파일 타임 결정)
```

### 7.2 동적 배열 단계별 변화

```
초기: capacity=4, size=0, data = [_, _, _, _]

append(10): size=1, [10, _, _, _]                  비용: 1
append(20): size=2, [10, 20, _, _]                 비용: 1
append(30): size=3, [10, 20, 30, _]                비용: 1
append(40): size=4, [10, 20, 30, 40]               비용: 1
append(50): size > capacity → 재할당!              비용: 4 (복사) + 1 = 5
  새 capacity = 8
  새 메모리: [10, 20, 30, 40, 50, _, _, _]
append(60): size=6, [10, 20, 30, 40, 50, 60, _, _]  비용: 1
...
```

평균 비용 (6 번 append) = (1+1+1+1+5+1) / 6 = 1.67 → **amortized O(1)** 의 직관.

### 7.3 중간 삽입 / 삭제 — O(N)

```
a = [10, 20, 30, 40, 50]
insert(2, 99):
  Step 1: 50 → 인덱스 5 로 shift
  Step 2: 40 → 인덱스 4
  Step 3: 30 → 인덱스 3
  Step 4: 인덱스 2 에 99 저장
결과: [10, 20, 99, 30, 40, 50]
복사 비용: O(N - i) — 끝에 가까울수록 빠름
```

---

## 8. 복잡도 종합

### 시간 / 공간

| 연산 | 정적 | 동적 (amortized) | 최악 | 캐시 미스 |
| --- | --- | --- | --- | --- |
| `a[i]` 접근 | O(1) | O(1) | O(1) | 0 (인접) |
| 끝에 append | ❌ | **O(1)** | O(N) (resize) | 0 |
| 앞에 insert | O(N) | O(N) | O(N) | O(N) |
| 중간 insert/del | O(N) | O(N) | O(N) | O(N) |
| 순차 순회 | O(N) | O(N) | O(N) | O(N/cache_line) |
| 검색 (값) | O(N) | O(N) | O(N) | O(N) |
| 검색 (정렬됨) | O(log N) | O(log N) | O(log N) | O(log N) |
| 정렬 | O(N log N) | O(N log N) | O(N²) (퀵 최악) | — |
| 공간 | O(N) | O(N) (max 2×) | — | — |

### Cache complexity (캐시 미스 횟수)

전통 Big-O 외에 **External Memory Model** 에서는 disk/cache 읽기 횟수도 분석:

- 배열 순회: `O(N / B)` cache miss (B = cache line size).
- 연결 리스트 순회: `O(N)` cache miss.
- 이진 검색 (배열): `O(log(N/B))` (대부분 캐시 안에서).
- 이진 트리 검색: `O(log N)` cache miss.

---

## 9. 의사 코드

생략 — 너무 간단. 자세한 건 [[#11-언어별-구현]] 참조.

---

## 10. 학술적 변형 (Advanced Variants)

### 10.1 Persistent Array (불변 배열)

함수형 언어 (Haskell, Clojure) 에서 사용. 갱신 시 새 버전 생성 + 공유.
- **Hash Array Mapped Trie (HAMT)**: 32-way 트리. Clojure `vector`.
- **Finger Tree**: 양 끝 O(1), 중간 O(log N).

### 10.2 Sparse Array (희소 배열)

대부분 0 또는 default 값일 때:
- **CSR (Compressed Sparse Row)** — 행렬에 자주 사용.
- **Dictionary of Keys (DOK)** — `{(i, j): value}` 만 저장.
- NumPy `scipy.sparse`, GPU 의 CSR 행렬.

### 10.3 Concurrent / Lock-free Array

- **Java `CopyOnWriteArrayList`** — 쓰기 시 복사. 읽기는 lock-free.
- **Atomic Array** — 각 원소가 atomic. lock-free 갱신.

### 10.4 Memory-mapped Array

- `mmap` 으로 거대 파일을 배열처럼. NumPy `memmap`.
- 메모리보다 큰 데이터 처리.

### 10.5 SIMD-aligned Array

캐시 라인 / SIMD 레지스터에 맞춰 정렬 (alignment) 된 배열:
- `_mm_malloc(size, 64)` — 64 byte 정렬.
- 캐시 라인 split 방지.

### 10.6 Cache-oblivious Array

블록 크기를 모르고도 캐시 효율 좋게 — Funnel sort / Van Emde Boas layout.

---

## 11. 언어별 구현 (11 언어)

(이전 버전과 동일 — 생략 가능. 핵심 11 언어 코드는 v.1.0.0 에 있음.)

### Python 3 (요약 — 본질)

```python
# Python list = PyObject* 의 동적 배열
# 내부 구조 (CPython):
#   ob_item: 포인터 배열
#   ob_size: 현재 크기
#   allocated: capacity

a = [1, 2, 3]       # heap 에 PyListObject 할당
a.append(4)          # O(1) amortized
x = a[2]             # O(1) — 포인터 역참조
a.sort()             # Timsort (안정, O(N log N))

# Numpy 는 진짜 연속 메모리 배열 (C 배열 wrapping)
import numpy as np
arr = np.array([1, 2, 3], dtype=np.int32)  # 12 byte 연속 메모리
arr.strides   # (4,) — 다음 원소까지의 byte 거리
```

전체 11 언어 구현은 분량상 v.1.0.0 참조. **요약 표**:

| 언어 | 동적 배열 | 문자열 mutability | 비고 |
| --- | --- | --- | --- |
| Python | `list` (PyObject* 배열) | immutable | NumPy 가 진짜 연속 |
| Java | `ArrayList<T>` (Object[]) | immutable + intern | `int[]` 은 primitive 배열 |
| C++ | `vector<T>` (연속 메모리) | mutable | SSO + small_vector |
| C | `int*` + `malloc/realloc` | mutable | 직접 관리 |
| JS | `Array` (sparse 가능) | immutable | V8 은 PackedSmi 최적화 |
| TS | JS 동일 + 타입 | — | — |
| Kotlin | `ArrayList` (JVM 기반) | immutable | `IntArray` primitive |
| Swift | `Array` (Copy-on-Write) | immutable (let) | Grapheme cluster 인식 |
| Go | `slice` (배열 + len + cap) | immutable | underlying array 공유 주의 |
| Rust | `Vec<T>` (소유권 + heap) | mutable | `&str` (slice) + `String` (owned) |
| C# | `List<T>` | immutable + intern | `Span<T>` 로 stack array |

전체 코드 예시는 [[#실습-코드-직접-동적-배열-구현]] 참조.

---

## 12. 핵심 패턴 / 변형

### (a) 투 포인터 (Two Pointers)

배열의 **양 끝** 또는 **느리게/빠르게** 두 인덱스 동시 이동. O(N).

```python
def two_sum_sorted(arr, target):
    lo, hi = 0, len(arr) - 1
    while lo < hi:
        s = arr[lo] + arr[hi]
        if s == target: return (lo, hi)
        elif s < target: lo += 1
        else: hi -= 1
    return None
```

**이론**: 정렬된 배열에서 단조성 (monotone) 이 보장될 때 적용.

### (b) 슬라이딩 윈도우

연속 부분 배열의 합 / 최댓값 / 길이. O(N).

```python
def max_sum(arr, k):
    window_sum = sum(arr[:k])
    best = window_sum
    for i in range(k, len(arr)):
        window_sum += arr[i] - arr[i-k]   # O(1) 갱신
        best = max(best, window_sum)
    return best
```

### (c) 누적합 (Prefix Sum)

구간 합 쿼리 O(1). 전처리 O(N).

```python
def build_prefix(arr):
    p = [0]
    for x in arr: p.append(p[-1] + x)
    return p

# range_sum(l, r) = p[r+1] - p[l]
```

**확장**:
- 2D 누적합 — 직사각형 영역 합 O(1).
- 차분 배열 (Difference Array) — 구간 갱신 O(1) + 최종 합 O(N).

### (d) 카데인 (Kadane) — DP 의 시작

최대 부분 배열 합. DP 의 가장 단순한 예.

```python
def max_subarray(arr):
    best = cur = arr[0]
    for x in arr[1:]:
        cur = max(x, cur + x)   # 점화식: dp[i] = max(x, dp[i-1] + x)
        best = max(best, cur)
    return best
```

### (e) Reservoir Sampling

스트림에서 균등 k개 샘플링 — 한 번의 패스 + O(k) 공간.

### (f) Quickselect — k번째 작은 원소

퀵 정렬의 partition 만 반복 → O(N) 평균.

### (g) Boyer-Moore Majority Vote

배열에서 과반수 원소를 O(N) 시간 + O(1) 공간으로.

---

## 13. 함정 / 안티패턴 (실무 + 코테)

### 함정 1 — Python `list * N` 의 얕은 복사

```python
matrix = [[0] * M] * N   # 잘못 — 모든 행이 같은 리스트 참조
matrix[0][0] = 1         # → matrix[1][0] 도 1
matrix = [[0] * M for _ in range(N)]   # 옳음
```

**이유**: `[[0]*M]` 은 한 리스트 객체. `* N` 은 그 참조를 N 번 복제.

### 함정 2 — 문자열 누적 += 의 O(N²)

```python
s = ""
for c in chars: s += c   # O(N²) — 매번 새 객체

s = "".join(chars)        # O(N)
```

**이유**: immutable. CPython 은 일부 최적화 (refcount=1 이면 in-place) 하지만
의존하지 말 것.

### 함정 3 — 인덱스 OOB

```python
for i in range(len(arr)):
    if arr[i+1] > arr[i]: ...   # i = len-1 일 때 OOB

for i in range(len(arr) - 1):    # 옳음
    if arr[i+1] > arr[i]: ...
```

### 함정 4 — Java `Arrays.asList` 의 immutable view

```java
List<Integer> list = Arrays.asList(arr);
list.add(4);   // UnsupportedOperationException
```

해결: `new ArrayList<>(Arrays.asList(arr))`.

### 함정 5 — JS `sort()` 의 문자열 비교

```javascript
[10, 2, 1].sort();   // ['1', '10', '2']
[10, 2, 1].sort((a, b) => a - b);
```

### 함정 6 — Go slice 의 underlying array 공유

```go
s1 := []int{1, 2, 3, 4, 5}
s2 := s1[1:3]
s2[0] = 99   // s1[1] 도 99
```

복사 원하면 `append([]int{}, s1[1:3]...)`.

### 함정 7 — Rust 의 owned vs borrowed

```rust
let s: String = "hello".to_string();
let slice: &str = &s[..3];      // 빌림
// drop(s);   // 그러면 slice 도 사용 불가
```

### 함정 8 — 큰 배열에 `in` 연산

```python
if x in big_list: ...     # O(N)
big_set = set(big_list)
if x in big_set: ...      # O(1)
```

### 함정 9 — 부동소수점 등호 비교

```python
if a == b: ...                    # 위험
if abs(a - b) < 1e-9: ...          # 안전
```

### 함정 10 — UTF-8 문자열을 byte 단위로 처리

```python
s = "안녕"
s[0]   # '안' — Python 은 코드 포인트 단위 (안전)

# 하지만 Go/C 는 byte
s = []byte("안녕")
s[0]   # 0xEC — UTF-8 첫 byte (의미 없음)
```

다국어 처리는 항상 **인코딩 명시 + 라이브러리 사용** (Python `unicodedata`, ICU).

---

## 14. 실전 풀이 절차

1. **자료구조 확정** — 정적 / 동적 / 정렬된 배열?
2. **인덱스 0/1 통일** — 입력 변환.
3. **메모리 추산** — N × sizeof(원소) ≤ 메모리 제한 / 2.
4. **연산 복잡도 매칭** — N=10^5 + 단순 패턴 → 패턴 (투포인터/슬라이딩/누적합) 적용.
5. **엣지** — N=0/1 / 음수 / 중복 / 동일값 / 큰 값.
6. **언어별 최적화** — Python `sys.stdin.readline` / Java `BufferedReader` / C++ `ios_base::sync_with_stdio(false)`.

---

## 15. 대표 문제 + 풀이

(v.1.0.0 의 6 대표 문제 + 풀이 코드 그대로 유지)

### Q1. Two Sum (LeetCode 1)
```python
def two_sum(nums, target):
    seen = {}
    for i, x in enumerate(nums):
        if target - x in seen: return [seen[target-x], i]
        seen[x] = i
```

### Q2. Maximum Subarray (Kadane)
```python
def max_subarray(nums):
    best = cur = nums[0]
    for x in nums[1:]:
        cur = max(x, cur + x)
        best = max(best, cur)
    return best
```

### Q3. Search in Rotated Sorted Array

```python
def search(nums, target):
    lo, hi = 0, len(nums) - 1
    while lo <= hi:
        mid = (lo + hi) // 2
        if nums[mid] == target: return mid
        if nums[lo] <= nums[mid]:
            if nums[lo] <= target < nums[mid]: hi = mid - 1
            else: lo = mid + 1
        else:
            if nums[mid] < target <= nums[hi]: lo = mid + 1
            else: hi = mid - 1
    return -1
```

### Q4. Trapping Rain Water

```python
def trap(height):
    if not height: return 0
    n = len(height)
    left_max = [0] * n; right_max = [0] * n
    left_max[0] = height[0]
    for i in range(1, n): left_max[i] = max(left_max[i-1], height[i])
    right_max[n-1] = height[n-1]
    for i in range(n-2, -1, -1): right_max[i] = max(right_max[i+1], height[i])
    return sum(min(left_max[i], right_max[i]) - height[i] for i in range(n))
```

### Q5. Sliding Window Maximum (Deque)

```python
from collections import deque
def max_sliding(nums, k):
    dq = deque()
    result = []
    for i, x in enumerate(nums):
        while dq and nums[dq[-1]] < x: dq.pop()
        dq.append(i)
        if dq[0] == i - k: dq.popleft()
        if i >= k - 1: result.append(nums[dq[0]])
    return result
```

### Q6. Longest Palindromic Substring

```python
def longest_palindrome(s):
    if not s: return ""
    start = end = 0
    for i in range(len(s)):
        l1 = expand(s, i, i)
        l2 = expand(s, i, i+1)
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

---

## 16. 연습 문제 (난이도별)

### 입문
- [백준 10818 최소 최대](https://www.acmicpc.net/problem/10818)
- [LeetCode 1 Two Sum](https://leetcode.com/problems/two-sum/)
- [LeetCode 27 Remove Element](https://leetcode.com/problems/remove-element/)

### 중급
- [백준 11659 구간 합 구하기 4](https://www.acmicpc.net/problem/11659) — 누적합
- [LeetCode 53 Maximum Subarray](https://leetcode.com/problems/maximum-subarray/)
- [LeetCode 121 Best Time to Buy/Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/)
- [LeetCode 209 Min Subarray Sum](https://leetcode.com/problems/minimum-size-subarray-sum/) — 슬라이딩

### 고급
- [LeetCode 42 Trapping Rain Water](https://leetcode.com/problems/trapping-rain-water/)
- [LeetCode 76 Min Window Substring](https://leetcode.com/problems/minimum-window-substring/)
- [LeetCode 239 Sliding Window Max](https://leetcode.com/problems/sliding-window-maximum/)

---

## 17. 실습 코드 — 직접 동적 배열 구현 + 분석

### Python — 직접 구현 + amortized 검증

```python
import ctypes

class DynamicArray:
    """동적 배열 직접 구현 + amortized 분석을 측정 가능하게."""
    def __init__(self):
        self._n = 0
        self._capacity = 1
        self._A = self._make_array(self._capacity)
        self.resize_count = 0   # 통계

    def __len__(self): return self._n

    def __getitem__(self, k):
        if not 0 <= k < self._n: raise IndexError
        return self._A[k]

    def append(self, obj):
        if self._n == self._capacity:
            self._resize(2 * self._capacity)
        self._A[self._n] = obj
        self._n += 1

    def _resize(self, c):
        self.resize_count += 1
        B = self._make_array(c)
        for k in range(self._n): B[k] = self._A[k]
        self._A = B
        self._capacity = c

    def _make_array(self, c):
        return (c * ctypes.py_object)()

# Amortized 측정
import time
arr = DynamicArray()
N = 1_000_000
start = time.time()
for i in range(N): arr.append(i)
elapsed = time.time() - start

print(f'N = {N}')
print(f'총 시간: {elapsed*1000:.1f} ms')
print(f'평균 시간: {elapsed*1000/N:.4f} ms / op')
print(f'Resize 횟수: {arr.resize_count}')
print(f'log2(N) = {N.bit_length()}')   # 약 20
# 결과: ~20번 resize. 평균 amortized O(1).
```

### Branch prediction 실험 (C++)

```cpp
#include <bits/stdc++.h>
#include <chrono>
using namespace std;
using namespace chrono;

int main() {
    const int N = 32768;
    vector<int> data(N);
    srand(0);
    for (auto& x : data) x = rand() % 256;
    
    // sort(data.begin(), data.end());   // 켜면 5배 빨라짐
    
    long long sum = 0;
    auto start = high_resolution_clock::now();
    for (int i = 0; i < 100000; i++) {
        for (int j = 0; j < N; j++) {
            if (data[j] >= 128) sum += data[j];
        }
    }
    auto elapsed = duration_cast<milliseconds>(high_resolution_clock::now() - start);
    cout << "Time: " << elapsed.count() << "ms, sum=" << sum << endl;
}
```

`sort` 주석 처리 vs 활성화 → 5-7배 시간 차이 측정 가능.

---

## 18. 학습 자료

### 책 (이론 깊이)
- **Introduction to Algorithms (CLRS) 4th** — Chapter 10 (Elementary Data Structures), 17 (Amortized Analysis)
- **The Algorithm Design Manual (Skiena)** — Chapter 3 (Data Structures)
- **Programming Pearls (Jon Bentley)** — Column 1-2 (배열의 깊이)
- **Computer Systems: A Programmer's Perspective (CSAPP)** — Chapter 6 (Memory Hierarchy)
- **What Every Programmer Should Know About Memory (Ulrich Drepper, 2007)** — [무료 PDF](https://akkadia.org/drepper/cpumemory.pdf) — 캐시 / SIMD 의 바이블

### 논문
- Liskov & Zilles (1974) — "Programming with abstract data types" (ADT 원논문)
- Pike & Thompson (1992) — "Hello World or Καλημέρα κόσμε or こんにちは 世界" (UTF-8 의 탄생)
- Bender et al. (2005) — "Cache-Oblivious Algorithms" (캐시 무관 알고리즘)

### 강의 / 영상
- [MIT 6.006 Lecture 2 - Data Structures](https://ocw.mit.edu/courses/6-006-introduction-to-algorithms-spring-2020/)
- [Stanford CS166 Data Structures](https://web.stanford.edu/class/cs166/)
- [동빈나 — 자료구조 기초](https://www.youtube.com/@dongbinna)

### 실측 도구
- [Compiler Explorer (godbolt.org)](https://godbolt.org/) — 어셈블리 출력
- [perf (Linux)](https://perf.wiki.kernel.org/) — 캐시 미스 / 분기 예측 측정
- [Intel VTune](https://www.intel.com/content/www/us/en/developer/tools/oneapi/vtune-profiler.html) — 깊은 프로파일

### 한국어
- "Do it! 자료구조와 함께 배우는 알고리즘 입문" (한빛미디어)
- "윤성우의 열혈 자료구조" (오렌지미디어) — C 기반 직접 구현
- 인프런 자료구조 강의

### Stack Overflow 의 명저답변
- [Why is processing a sorted array faster than processing an unsorted array?](https://stackoverflow.com/questions/11227809/) — 분기 예측의 고전
- [What is "cache-friendly" code?](https://stackoverflow.com/questions/16699247/) — 캐시 친화 코드

---

## 19. 관련

- [[../linked-lists/linked-lists]] — 동적 크기 + O(1) 삽입 (다음 노트)
- [[../hash-tables/hash-tables]] — O(1) 키 매핑 — 배열 + 해시 함수
- [[../../algorithm/sorting/sorting]] — 배열 정렬 알고리즘 사전
- [[../../algorithm/binary-search/binary-search]] — 정렬된 배열의 O(log N) 검색
- [[../../computer-architecture/computer-architecture]] — 메모리 계층 / 캐시 / CPU (작성 예정)
- [[../data-structure|↑ data-structure 인덱스]]
