---
title: "Bloom Filter — Probabilistic Set"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:10:00+09:00
tags:
  - data-structure
  - bloom-filter
  - probabilistic
---

# Bloom Filter — Probabilistic Set

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 동작 / FP 율 / 변형 |

**[[advanced|↑ Advanced]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

**작은 공간 + 빠른 멤버십 검사** — "없음" 100% 정확 / "있음" 오답 가능. 1970 Burton Bloom.

---

## 2. 동작

### 구조
- bit array (크기 m)
- k 개 hash 함수

### Insert
```
hash1("apple") = 3
hash2("apple") = 17
hash3("apple") = 42

→ bit[3], bit[17], bit[42] = 1
```

### Check
```
"apple":
  hash1, hash2, hash3 모두 1 → "있을 수도 있음"
  하나라도 0 → "절대 없음"
```

---

## 3. False Positive — 가능

```
"banana" → bits 3, 17, 42 (다른 hash 가 같은 비트 셋팅 했을 수도)
→ "있을 수도 있음" 응답 (실제는 없음)
```

### 절대 False Negative 없음
- 한 번 insert → 모든 비트 set
- 그 후 check → 항상 1 → "있음"

---

## 4. False Positive 율

### 공식
```
p = (1 - e^(-kn/m))^k
```

- n — 원소 수
- m — bit 수
- k — hash 함수 수

### 최적 k
```
k = (m/n) × ln(2)
```

### 예
- 10M 원소, FP 1%
  - m ≈ 96M bit (12 MB)
  - k ≈ 7

### Trade-off
- m ↑ — FP ↓ (메모리 ↑)
- k 너무 많음 — 모든 bit 1 으로 가득

---

## 5. 구현

```python
import mmh3
from bitarray import bitarray

class BloomFilter:
    def __init__(self, n, p):
        # n: expected items, p: target FP rate
        import math
        self.m = int(-(n * math.log(p)) / (math.log(2) ** 2))
        self.k = int((self.m / n) * math.log(2))
        self.bits = bitarray(self.m)
        self.bits.setall(False)
    
    def add(self, item):
        for i in range(self.k):
            idx = mmh3.hash(item, i) % self.m
            self.bits[idx] = True
    
    def __contains__(self, item):
        for i in range(self.k):
            idx = mmh3.hash(item, i) % self.m
            if not self.bits[idx]: return False
        return True    # 있을 수도 있음
```

---

## 6. 시간 / 공간

| | 시간 |
| --- | --- |
| Insert | O(k) |
| Lookup | O(k) |

| 공간 | O(m) bit |

### Hash table 과 비교
- Hash table — 10M string × 50 byte ≈ 500 MB
- Bloom filter — 12 MB (1% FP)
- 약 40배 절약

---

## 7. 응용

### 7.1 Cache miss 회피
- Cache 에 없으면 — DB 조회 X (Bloom 검사 후)
- "이 key 는 DB 에 절대 없음" — 비싼 query 절약

### 7.2 Cassandra / RocksDB
- SSTable 검색 전 Bloom — 99% 의 disk read 절약

### 7.3 BigTable / HBase
- 비슷

### 7.4 URL spam check
- 알려진 spam URL — Bloom
- "절대 spam X" 빠른 응답

### 7.5 Crypto / VPN
- IP 차단 list
- Tor — 다음 hop 후보

### 7.6 Set difference
- 양쪽 시스템 의 데이터 — 다른 것만 동기

### 7.7 LSM Tree
- Memtable + SSTables
- Bloom — 어느 SSTable 인지 빠르게 (자세히 [[lsm-tree]])

---

## 8. 변형

### Counting Bloom Filter
- bit → counter
- Delete 가능 (decrement)
- 메모리 ↑

### Counting Quotient Filter / Cuckoo Filter
- Bloom 의 모던 대체
- Delete 가능 + locality 좋음

### Scalable Bloom Filter
- 가득 차면 — 새 Bloom 추가
- 누적 FP 가 알맞

### Compressed Bloom
- Bit array compress

### Stable Bloom (streaming)
- 옛 항목 자동 만료
- 데이터 stream

---

## 9. Cuckoo Filter

### 정의
- Cuckoo Hashing 기반
- Fingerprint 저장 (key 의 짧은 hash)
- Delete 가능
- Cache locality 좋음

### 효과
- 같은 FP 율 — 메모리 비슷
- Delete OK
- 속도 빠름 (cache)

자세히 → [[cuckoo-hashing]]

---

## 10. HyperLogLog — Cardinality

### 정의
- "고유 원소 수" estimate
- 매우 작은 메모리 (KB)

### 사용
- Redis HLL
- 유니크 visitor 수
- 데이터 분석

---

## 11. 함정

### 함정 1 — Delete 안 됨
Standard Bloom — delete X. Counting Bloom 사용.

### 함정 2 — FP 율의 계획
n 잘못 추정 → p 다름. 여유 두기.

### 함정 3 — Hash 의 독립성
k 개 hash — 독립 (실제 — k 개 다른 seed).

### 함정 4 — Memory bandwidth
큰 m — k 번 random access. Cache miss.
Block Bloom Filter — 한 cache line 안.

### 함정 5 — 같은 key 여러 번 insert
중복 — 효과 없음 (이미 bit set).

### 함정 6 — 만료 / TTL
Standard Bloom — 만료 X. Stable / Sliding Bloom.

---

## 12. 학습 자료

- "Space/Time Trade-offs in Hash Coding with Allowable Errors" — Bloom 1970
- "Cuckoo Filter" — Fan, 2014
- Redis Bloom module docs

---

## 13. 관련

- [[advanced]] — Hub
- [[cuckoo-hashing]] — 대체
- [[lsm-tree]] — RocksDB 사용
- [[../hash-tables/hash-functions]]
