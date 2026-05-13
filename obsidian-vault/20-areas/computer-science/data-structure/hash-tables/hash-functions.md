---
title: "Hash Functions — 해시 함수"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T14:00:00+09:00
tags:
  - data-structure
  - hash
---

# Hash Functions — 해시 함수

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 종류 / 균등성 / DoS |

**[[hash-tables|↑ Hash Tables]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

임의 입력 → 고정 크기 출력. 좋은 해시 — **균등 분산 + 빠름 + 결정적**.

---

## 2. 좋은 해시의 조건

1. **결정적** — 같은 입력 → 같은 출력
2. **균등 분산** — 모든 bucket 동등 확률
3. **빠름** — O(1) 또는 O(key size)
4. **avalanche** — 1 비트 변경 → 출력 절반 변경

---

## 3. 비암호 해시 — 일반 hash table 용

### FNV (Fowler-Noll-Vo, 1991)
```c
hash = FNV_OFFSET;
for (byte in key):
    hash ^= byte;
    hash *= FNV_PRIME;
```

- 단순, 빠름
- 옛 hash table

### MurmurHash (Austin Appleby, 2008)
- 빠름 / 균등성 좋음
- HashMap (Java), Redis, Cassandra

### CityHash (Google, 2011)
- 큰 string 빠름
- AES-NI 활용

### xxHash (2012)
- 가장 빠름 (cycle/byte 0.3)
- LZ4 / zstd

### SipHash (Jean-Philippe Aumasson, 2012)
- 짧고 키 기반 — DoS 방어
- Python / Ruby / Rust hashmap 기본

---

## 4. 암호 해시 — 보안 / 무결성

### MD5 (1991)
- 깨짐 (collision 발견 2004)
- 무결성만 (보안 X)

### SHA-1 (1995)
- 깨짐 (Google 2017)
- Git 의 hash (마이그레이션 중)

### SHA-2 (SHA-256, SHA-512)
- 안전 — 현재 표준
- Bitcoin / TLS / GPG

### SHA-3 (Keccak, 2015)
- 다른 구조 (sponge)
- SHA-2 의 대체 옵션

### BLAKE3 (2020)
- 매우 빠름 (병렬화)
- 보안 OK

자세히 → [[../../network/tls-ssl/cipher-suites]]

---

## 5. 해시 vs 암호 해시

| | Hash | Cryptographic Hash |
| --- | --- | --- |
| **속도** | 매우 빠름 (ns) | 느림 (μs+) |
| **충돌** | OK | 매우 어려움 |
| **출력 크기** | 32-64 bit | 128-512 bit |
| **DoS** | 약함 (random seed 필요) | 강함 |
| **사용** | Hash table | 보안 / 무결성 |

---

## 6. Hash randomization — DoS 방어

### 문제 — Hash collision attack
- 공격자가 같은 hash 의 key 모음 — 모든 chain 한 bucket 으로
- O(N²) 부담 (수 분 → 수 시간)

### 사례
- 2003 — Perl / Java / Python 의 Hash table DoS

### 해결 — Hash randomization
- 시작 시 random seed
- 같은 key — 매번 실행 별 다른 hash

### Python
```bash
PYTHONHASHSEED=0 python    # 결정적 (디버그)
PYTHONHASHSEED=random python   # default
```

### Java
- ConcurrentHashMap — random
- HashMap — 옛 deterministic, treeify (chain → tree) for 8+

---

## 7. 일관성 — Modulo

### 단순
```
bucket = hash(key) % size
```

### Size = prime — 균등
- 비-prime — 패턴 있는 hash 가 한 bucket 에 몰림

### Power of 2 — 빠른 modulo
- `hash & (size - 1)` (size = 2^k)
- Java HashMap 사용

### 단점
- 하위 비트만 사용 — 좋은 hash 필요
- Fibonacci hashing 권장 (가져)

---

## 8. Fibonacci hashing

```
bucket = (hash * 2^k * golden_ratio) >> shift
       = (hash * 0x9E3779B9) >> (32 - k)
```

### 효과
- 모든 비트 사용
- 균등 좋음
- 빠른 multiply

### 사용
- Rust HashMap
- Some C++ stdlib

---

## 9. 언어별 해시 함수

### Python
- str / bytes — SipHash (default)
- int — identity (자기 자신)
- float / complex — 특수
- tuple — XOR / multiply of elements

### Java
- `String.hashCode()` — 31 multiply
- `Object.hashCode()` — identity (memory address-based)
- `Objects.hash(a, b, c)` — combine

### Go
- map의 hash — runtime 자체 (구현 비공개, fast)
- AES-NI 활용 (intel cpu)

### Rust
- `std::collections::HashMap` — SipHash 1-3 (default)
- `ahash` / `fxhash` — 더 빠른 옵션
- Hash trait — 커스텀

### C++
- `std::hash<T>` — implementation defined
- 보통 FNV / Murmur 변형

---

## 10. 사용자 정의 해시

### 좋은 hash 조건
1. 멤버 모두 사용
2. 순서 / 대칭성 고려
3. 곱셈 + 누적 (XOR / 더하기보다)

### 예 — Pair
```python
def hash_pair(a, b):
    return hash((a, b))    # Python tuple

# 자체:
def hash_pair(a, b):
    return ((hash(a) * 31) + hash(b)) & 0xFFFFFFFF
```

### Java `Objects.hash(...)`
```java
@Override
public int hashCode() {
    return Objects.hash(name, age, email);
}
```

### Equality contract
- `equals → hash 같다` (necessary)
- `hash 같다 → equals` (X — collision)

---

## 11. Hash distribution 측정

### Chi-squared test
- bucket 별 count 의 분산
- 균등 분산 — chi² 작음

### Avalanche test
- 1 bit 변경 — 출력 50% 변경?

### SMHasher
- Hash 함수 테스트 도구
- MurmurHash, xxHash 인증

---

## 12. 함정

### 함정 1 — hash 만 일치 ≠ key 일치
Collision. 비교 필수.

### 함정 2 — Mutable key
key 변경 → hash 변경 → lookup 못 함. Immutable / hash 고정.

### 함정 3 — Float 의 hash
NaN ≠ NaN — hash 다름 가능. 특수 처리.

### 함정 4 — Identity hash 위험
Memory address — 매 실행 다름. 직렬화 X.

### 함정 5 — DoS
Random seed 없으면 — 공격 가능. 모던 언어 — 자동.

### 함정 6 — 작은 hash space
8 bit hash — 256 bucket. 큰 N — 매우 많은 collision. 32/64 bit.

### 함정 7 — equals 와 hashCode 의 불일치
Java contract 위반 — HashMap 못 찾음.

---

## 13. 학습 자료

- "Hash Functions and the Pigeonhole Principle"
- SMHasher GitHub
- Python `__hash__` docs
- "Fibonacci Hashing" — Probably Dance 블로그

---

## 14. 관련

- [[hash-tables]] — Hub
- [[collision-resolution]] — Chaining / Open Addressing
- [[../../network/topics/consistent-hashing]] — 분산 해시
