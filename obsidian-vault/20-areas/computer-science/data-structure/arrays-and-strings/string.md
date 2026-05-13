---
title: "String — 문자열의 자료구조"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T11:05:00+09:00
tags:
  - data-structure
  - string
  - unicode
---

# String — 문자열의 자료구조

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | immutable / interning / Unicode |

**[[arrays-and-strings|↑ Arrays & Strings]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

**문자의 sequence** — 보통 byte array + 인코딩 (UTF-8 / UTF-16 / ASCII).
언어 별 — immutable (Python / Java) / mutable (C / Go bytes).

---

## 2. 핵심 — 메모리 표현

### C 의 문자열
```c
char s[] = "Hello";    // ['H','e','l','l','o','\0']
                       // null-terminated
```

- 끝 — `\0` (NULL byte)
- strlen — O(N) (`\0` 까지)
- 안전 — buffer overflow 위험

### Pascal-style / 모던
```
[ length: 5 ][ H ][ e ][ l ][ l ][ o ]
```

- length prefix
- O(1) length

### Rope / String builder
- 연속 메모리 X — tree 또는 chunk
- 큰 문자열 / 잦은 concat 친화

---

## 3. Immutable vs Mutable

### Immutable (Python / Java / JS / C# / Swift)
```python
s = "Hello"
s += " World"     # 새 문자열 — 옛 s 그대로
```

#### 효과
- Thread-safe
- Hashing / dict key 가능
- Interning 가능

#### 단점
- Concat 비싸짐 (매번 새 문자열)
- 큰 string + 작은 변경 — 비효율

### Mutable (StringBuilder / bytes)
```java
StringBuilder sb = new StringBuilder();
sb.append("Hello");
sb.append(" World");   // 같은 buffer
String result = sb.toString();
```

```python
# bytes (mutable)
b = bytearray(b"Hello")
b.append(ord('!'))
```

---

## 4. Concat 의 O(N²) 함정

### 잘못된
```python
s = ""
for i in range(N):
    s += str(i)        # 매번 새 string — O(N²)
```

### 올바른
```python
parts = []
for i in range(N):
    parts.append(str(i))
s = "".join(parts)     # O(N)
```

### CPython 의 최적화
- 단일 reference 일 때 — in-place (옵션)
- 신뢰 X — 명시 join

### Java / C# 의 StringBuilder
- 명시적 mutable

---

## 5. String interning

### 정의
- 같은 내용의 string — 같은 메모리 객체

### Python
```python
a = "hello"
b = "hello"
a is b           # True (interning)

a = "hello world!"
b = "hello world!"
a is b           # 구현 따름 (True or False)
```

#### 자동 interning (Python)
- 짧은 / identifier-like — 자동
- 명시 — `sys.intern(s)`

### Java
```java
String a = "hello";
String b = "hello";
a == b;          // true (string pool)

String c = new String("hello");
a == c;          // false
a == c.intern(); // true
```

### 효과
- 메모리 절약 (반복 string)
- == 비교 빠름 (pointer 비교)
- 단점 — 큰 interning pool

---

## 6. Hash code

### String 의 hash
```python
hash("hello")    # 큰 정수
```

### Java 의 String hashCode
```java
public int hashCode() {
    int h = 0;
    for (char c : value) {
        h = 31 * h + c;
    }
    return h;
}
```

→ Java — 31 (소수, multiply optimization)

### Python 의 hash randomization (security)
- PYTHONHASHSEED — 매 실행 다른 seed
- DoS 방어 (hash collision attack)

---

## 7. Unicode

### UTF-8 (가변)
```
ASCII (0-127):       1 byte
2 byte chars (Latin): 2 byte
3 byte chars (한글): 3 byte
4 byte chars (이모지): 4 byte
```

#### 장점
- ASCII backward compatible
- Variable size (작은 → 작게)

#### 단점
- O(1) index 어려움 (byte != char)

### UTF-16 (Java / JS / Windows / .NET)
```
BMP (basic 0-FFFF):  2 byte
Supplementary:        4 byte (surrogate pair)
```

#### 단점
- 이모지 surrogate pair — index 위험
- 같은 문자열 — 다른 byte 수

### UTF-32 (rare)
- 4 byte 고정
- 메모리 낭비
- O(1) index

### Python 의 string
```python
s = "안녕"
len(s)        # 2 (Python 3 — Unicode code points)

# Python 3 — UCS-1/2/4 자동
# Max code point 따라 1/2/4 byte 선택
```

### Go의 string
```go
s := "안녕"
len(s)            // 6 (byte 수 — UTF-8)
utf8.RuneCountInString(s)  // 2 (실제 character)

for i, r := range s {
    // i: byte index, r: rune (Unicode code point)
}
```

### Rust 의 String / str
```rust
let s = String::from("안녕");
s.len();              // 6 (byte)
s.chars().count();    // 2 (char)
// s[0] — 컴파일 에러 (byte index 위험)
```

---

## 8. Grapheme clusters

### 한 character == 한 code point?
**X.** Unicode 의 grapheme cluster:

```
"é" — 다양:
1. U+00E9 (single)
2. U+0065 (e) + U+0301 (combining acute) (composed)

"🇰🇷" — 한 국기, 2 code point (KR)

"👨‍👩‍👧‍👦" — 한 가족 이모지, 7 code point (ZWJ joined)
```

### 안전한 처리
- ICU / unicode-segmentation crate
- `str.normalize` (NFC / NFD)

---

## 9. 흔한 연산 — 시간 복잡도

| 연산 | C string | Python | Java | Go |
| --- | --- | --- | --- | --- |
| `len` | O(N) | O(1) | O(1) | O(1) |
| `s[i]` (byte) | O(1) | O(1) | O(1) | O(1) |
| `s[i]` (char) | O(N) | O(1) | O(N) (UTF-16) | O(N) (UTF-8) |
| `s + t` | O(N) new | O(N) new | O(N) new | O(N) new |
| `substring(i, j)` | O(N) | O(N) | O(1) (옛) | O(1) (view) |
| `find` (naive) | O(N×M) | O(N×M) | O(N×M) | O(N×M) |
| `find` (KMP / BMH) | O(N+M) | (CPython opt) | (varied) | (varied) |

### Java 의 substring 변화
- 옛 (Java 6) — O(1) (share char[])
- 새 (Java 7+) — O(N) (copy) — memory leak 방지

---

## 10. String search 알고리즘

### Naive
- O(N×M)

### KMP (Knuth-Morris-Pratt)
- O(N+M)
- Failure function

### Boyer-Moore
- O(N) 평균 (큰 alphabet)
- 옛 grep

### Rabin-Karp
- Rolling hash
- 평균 O(N+M)

### Aho-Corasick
- 여러 pattern 동시
- 사용 — antivirus / NLP

자세히 → algorithm/string-matching

---

## 11. 함정

### 함정 1 — `+=` 의 O(N²)
Loop 의 string concat — `join` 또는 StringBuilder.

### 함정 2 — Encoding 가정
파일 / 네트워크 — encoding 확인. UTF-8 default.

### 함정 3 — `==` vs `equals`
Java — `s1 == s2` reference 비교 (보통 X). `s1.equals(s2)`.

### 함정 4 — Char index 의 의미
Python 3 — code point. Java — UTF-16 code unit. 다름.

### 함정 5 — 다국어 정렬
ASCII 정렬 ≠ locale 정렬. Collator (ICU) 필요.

### 함정 6 — Substring 의 메모리 leak
옛 Java — 큰 string 의 substring 이 원본 char[] 참조 (보존).

### 함정 7 — Reverse 의 grapheme
"abc🇰🇷" reverse — surrogate pair 깨짐. unicode-segmentation.

---

## 12. 학습 자료

- "Programming Pearls" (Bentley) — string ch
- Joel Spolsky — "The Absolute Minimum Every Software Developer Must Know About Unicode"
- Unicode.org docs
- Python docs — text vs bytes

---

## 13. 관련

- [[arrays-and-strings]] — Hub
- [[dynamic-array]] — 비슷한 메모리 모델
- [[../tries/tries]] — String prefix
