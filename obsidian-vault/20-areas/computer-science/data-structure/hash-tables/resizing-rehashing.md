---
title: "Resizing / Rehashing — 크기 조정"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T14:10:00+09:00
tags:
  - data-structure
  - hash
  - resize
---

# Resizing / Rehashing

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Load factor / Incremental / Amortized |

**[[hash-tables|↑ Hash Tables]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

Hash table 의 **load factor 가 한계 도달 → 크기 2배 + 모든 key 재 hashing**. Amortized O(1) 의 핵심.

---

## 2. 왜 Resize?

### Load factor 너무 높음
- Chaining — chain 길어짐 → O(N)
- Open addressing — probing 길어짐 → O(N)
- 일정 한도 (0.75) 도달 시 — 2배 확장

### Load factor 너무 낮음
- 메모리 낭비
- 일부 — shrink (0.1 이하 등)

---

## 3. Load Factor

### 정의
```
α = number of entries / capacity
```

### 권장
| | Load Factor |
| --- | --- |
| Chaining | 0.75-1.5 |
| Open Addressing | 0.5-0.7 |
| SwissTable | 0.875 |
| Python dict | < 2/3 |
| Java HashMap | 0.75 (default) |

---

## 4. Resize 절차

### 기본
```
1. 새 capacity = 2 * 옛 capacity
2. 새 bucket array 할당
3. 모든 옛 entry 의 key 를 다시 hashing
4. 새 위치에 저장
5. 옛 메모리 해제
```

### 비용
- O(N) — 모든 entry
- Resize 가 자주 안 일어남 → amortized O(1)

---

## 5. Amortized 분석

### 가정
- N entries → 어느 정도 resize 발생?
- size 1 → 2 → 4 → 8 → ... → 2^k

### 총 작업
```
1 + 2 + 4 + 8 + ... + 2^k = 2^(k+1) - 1 ≈ 2N
```

### Per operation — amortized O(1)
- N insert → 총 2N 작업 → 각 insert O(1) 평균

---

## 6. Incremental Rehashing

### 문제
- 큰 hash table — resize 1회 → multi-second pause
- Real-time / 고가용성 X

### 해결 — Incremental
- 한 번에 다 안 함
- 각 lookup / insert — 일부 (예: 1-2 bucket) 옮김

### Go map
- Resize 시 — 매 access 마다 1-2 bucket evacuation
- 점진적 — 다음 resize 까지 끝남

### Redis
- Background — 1ms 마다 일부
- Dictionary 의 dual table

---

## 7. Shrink — 줄이기

### 대부분 — 자동 shrink 안 함
- Memory 다시 차지
- Delete 빈번 — load factor 매우 낮음

### 명시 shrink
- Python — 일부 자동 (1/16 이하)
- Java — clear() / 새 객체 권장
- Go — 자동 X (전체 재할당)

### Tradeoff
- Shrink — resize 비용
- 안 함 — memory 낭비

---

## 8. Robin Hood / SwissTable 의 효율

### SwissTable
- Rehashing 빠름 (SIMD)
- Group 단위 처리

### Robin Hood
- Cluster 일관성 → cache 친화 rehash

---

## 9. Hopscotch Hashing

### 정의
- Open addressing 변형
- 각 key — 자기 home bucket 의 H 거리 안에 (H=32)
- Insert 시 — swap 으로 가까이 유지

### 효과
- 매우 cache 친화
- 빠른 lookup

### 사용
- ConcurrentHopscotchHashMap (Java 일부)

---

## 10. Concurrent Resize

### 문제
- Resize 중 — 다른 thread 의 lookup?

### Java `ConcurrentHashMap`
- Segment-based (Java 7) — 옛
- CAS + bin-level lock (Java 8+)
- Resize — helper thread 도 참여 가능

### Lock-free
- 복잡 — 일관성 보장 어려움

---

## 11. DoS / Resize 사고

### Hash flooding
- 공격자 — collision 많은 key 모음
- Insert N → O(N²) — resize 도 무용

### 방어
- Hash randomization (per-process seed)
- Treeify (chain → tree) — O(N log N) 최악

---

## 12. 시간 별 분포

### Resize 분포
```
Operations:  | | | | | | | | | | | | | | | | | | |
Resize:      |                |                  |
             O(N)             O(2N)
             ↑ pause
```

### Tail latency
- p99 가 resize 시 큼
- Incremental — p99 작음 (분산)

---

## 13. 함정

### 함정 1 — Iterator invalidation
Resize 중 — iterator 가리키는 entry 위치 변경. ConcurrentModificationException.

### 함정 2 — Resize 시점 예측
N 미리 알면 — `new HashMap<>(N)` 으로 미리 할당. resize 회피.

### 함정 3 — Real-time 환경의 resize
Game / trading — resize pause 위험. 미리 sized.

### 함정 4 — Shrink 안 함의 memory bloat
큰 burst 후 — memory 유지. 새 HashMap 만들기.

### 함정 5 — Power of 2 size 필요
빠른 modulo — 2^k. 일반 prime 보다 균등성 약함 (hash 좋아야).

### 함정 6 — Concurrent — resize 보존
Resize 중 read — 옛 vs 새 table? CAS / fence.

---

## 14. 학습 자료

- CLRS Ch 17 (amortized)
- Java 8 HashMap source
- Go runtime/map.go
- "Don't Modify the Linked List of a Concurrent HashMap" — anti-pattern

---

## 15. 관련

- [[hash-tables]] — Hub
- [[hash-functions]] / [[collision-resolution]]
- [[../arrays-and-strings/dynamic-array]] — 비슷한 amortized
