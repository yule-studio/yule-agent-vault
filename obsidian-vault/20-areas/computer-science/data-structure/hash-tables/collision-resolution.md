---
title: "Collision Resolution — Chaining vs Open Addressing"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T14:05:00+09:00
tags:
  - data-structure
  - hash
  - collision
---

# Collision Resolution

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Chaining / Open Addressing / Robin Hood |

**[[hash-tables|↑ Hash Tables]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

여러 key 가 같은 bucket — **해결 방법**:
- **Chaining** — bucket 별 linked list / tree
- **Open Addressing** — 다른 빈 bucket 찾기

---

## 2. Separate Chaining

### 구조
```
bucket[0] → [k1:v1] → [k4:v4]
bucket[1] → NULL
bucket[2] → [k2:v2] → [k5:v5] → [k7:v7]
```

### 동작
- Insert — chain head 에 추가
- Search — chain 순회
- Delete — chain 에서 제거

### 평균 chain 길이
- load factor α = n / size
- 평균 chain — α
- 평균 search — 1 + α/2 (균등 분산 가정)

### 장점
- Load factor > 1 OK
- Delete 쉬움
- 메모리 효율 좋음 (load 낮을 때)

### 단점
- Linked list — cache 미스
- 포인터 overhead

### Java HashMap (Java 8+)
- Chain 이 길어지면 (>= 8) — Red-Black Tree
- 최악 O(log N)

자세히 → [[../trees/red-black-tree]] (TBD)

---

## 3. Open Addressing

### 구조
```
bucket[0] = [k1:v1]
bucket[1] = [k2:v2]   ← collision: k2 hash = 0 → probe to 1
bucket[2] = [k3:v3]
bucket[3] = empty
```

### Insert
1. hash(key) → bucket i
2. bucket i 비어있으면 — 저장
3. 차 있으면 — probing (다음 bucket 시도)

### Search
1. hash(key) → bucket i
2. 같은 key? — 반환
3. 다른 key — probing 따라 계속
4. 빈 bucket 만나면 — 없음

### Delete — 어려움
- 빈 bucket 만들면 — search 끊김
- "Tombstone" 표시 (삭제됨, 그러나 search 시 skip)

---

## 4. Probing 전략

### Linear Probing
```
probe(i, j) = (hash(i) + j) % size
```

- 단순
- Primary clustering — 한 cluster 길어짐

### Quadratic Probing
```
probe(i, j) = (hash(i) + j²) % size
```

- Primary clustering ↓
- Secondary clustering 가능

### Double Hashing
```
probe(i, j) = (hash1(i) + j × hash2(i)) % size
```

- 가장 균등
- 두 hash 계산

### Cuckoo Hashing
- 2 hash 함수 사용
- 충돌 시 — 옛 key 를 두 번째 hash 의 bucket 으로 이동
- O(1) worst-case lookup
- Rehash 비싸

자세히 → [[../advanced/cuckoo-hashing]] (TBD)

---

## 5. Robin Hood Hashing

### 정의
- Open addressing 변형
- 더 많이 probing 한 key 가 — 적게 probing 한 key 의 자리 take ("riches from rich")

### 효과
- 평균 probing 균등
- Cache 친화

### 사용
- Rust HashMap (옛, 현재는 SwissTable)

---

## 6. SwissTable (Google, 2017)

### 정의
- Open addressing + SIMD
- Group 16 bucket — SSE2 instruction 으로 한꺼번에 검색

### 흐름
- hash → metadata (7 bit) + index
- Probe — 16 bucket group 의 metadata 비교 (1 SSE instruction)
- 매치 가능한 위치 — 비교

### 사용
- Google `absl::flat_hash_map`
- Rust HashMap (current)
- Go 1.24+ (대체)

### 효과
- 1-2 cache line 만 접근
- Linear probing 의 cluster 한계 극복

---

## 7. Chaining vs Open Addressing

| | Chaining | Open Addressing |
| --- | --- | --- |
| **Load factor** | > 1 OK | < 0.7 권장 |
| **Cache** | 나쁨 (linked) | 좋음 (연속) |
| **Memory** | 포인터 overhead | 알맞 |
| **Delete** | 쉬움 | 어려움 (tombstone) |
| **Resize** | 비슷 | 비슷 |
| **Worst case** | 좋은 hash → O(log N) (tree) | Bad — O(N) |

### 모던 선호 — Open Addressing
- Cache locality
- SIMD 활용
- 모던 hash (좋은 randomization)

---

## 8. Java HashMap 의 구현 (Java 8+)

### Chaining + Tree
```
bucket[0] → entry1 → entry2 → entry3 → ... → entry8 → tree-bucket
                                              ↑ TREEIFY_THRESHOLD = 8
```

### Resize
- Load factor 0.75 — capacity 2배 → rehash

### Tree → Chain (UNTREEIFY)
- Chain 짧아지면 (< 6) — 다시 linked list

---

## 9. Python dict (3.6+)

### Compact dict (PEP 468)
- Insertion order 유지
- 2 tables — sparse (index) + dense (entries)

```
indices:  [_, _, 0, _, 1, 2, _, _]
entries:  [
    (hash, key1, val1),
    (hash, key2, val2),
    (hash, key3, val3)
]
```

### Open addressing — perturbed probing
```python
i = hash(key) & mask
while occupied:
    perturb >>= 5
    i = (5 * i + 1 + perturb) & mask
```

### Resize
- 1/3 이하 또는 2/3 이상 → resize

---

## 10. Go map

### Bucket-based open addressing
- 한 bucket — 8 entries
- 가득 시 — overflow bucket linked

### Load factor — 6.5 평균 / bucket
- 8 까지 OK, 더 가면 grow

### Incremental resize
- 한 번에 다 안 옮기고 — 점진적 (lookup 마다 일부)

---

## 11. Rust HashMap

### SwissTable 기반 (hashbrown)
- 매우 빠름
- SipHash 1-3 default
- 옵션 — `ahash` (faster, less DoS resistant)

---

## 12. 함정

### 함정 1 — Bad hash 의 worst case
모든 key 한 bucket → O(N). 좋은 hash 필수.

### 함정 2 — Resize 의 latency spike
큰 hash table — resize 시 stutter. Incremental resize.

### 함정 3 — Tombstone 누적
Open addressing — 많은 delete 후 — tombstone 많음. Rehash.

### 함정 4 — Iteration 의 정의되지 않은 순서
대부분 — 정의 X (Python 3.6+ exception).
모던 — insertion order 또는 random.

### 함정 5 — Concurrent
대부분 — thread-unsafe. ConcurrentHashMap (Java) / Mutex.

### 함정 6 — Load factor 너무 높음
0.9+ — collision 폭발. 0.5-0.75 권장.

---

## 13. 학습 자료

- "Hash Tables" — CLRS Ch 11
- "SwissTable" Google Tech Talk (2017)
- "Robin Hood Hashing" — Pedro Celis 박사논문
- Java 8 HashMap source

---

## 14. 관련

- [[hash-tables]] — Hub
- [[hash-functions]] — 해시 함수
- [[resizing-rehashing]] — 크기 조정
- [[../trees/red-black-tree]] (TBD) — Java HashMap treeify
