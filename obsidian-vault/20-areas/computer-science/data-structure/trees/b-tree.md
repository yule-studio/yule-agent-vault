---
title: "B-Tree / B+Tree — DB 인덱스의 표준"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T16:20:00+09:00
tags:
  - data-structure
  - b-tree
  - database
---

# B-Tree / B+Tree

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | B-tree / B+tree / DB index |

**[[trees|↑ Trees]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

**Rudolf Bayer + Edward McCreight (1972)** — 한 노드에 여러 키. Disk 친화 — 한 노드 = 한 disk page. DB index 의 표준.

---

## 2. 왜 B-Tree?

### Disk I/O 의 비용
- Disk 의 random access — 10 ms (HDD) / 0.1 ms (SSD)
- Memory — 100 ns
- Disk read 1 번 — 매우 비싸

### B-Tree
- 한 노드 = 한 page (4 KB / 16 KB)
- 한 disk read 에 많은 키 검사
- 트리 높이 — log_t(N) (t = 한 노드 키 수)

### 예
- N = 10^9, t = 100 → 높이 4-5
- 5 disk read 로 검색

---

## 3. 구조

### B-Tree of degree t
- 각 노드 — 최대 2t-1 키, 2t 자식
- 각 노드 (root 제외) — 최소 t-1 키, t 자식
- 모든 leaf — 같은 깊이 (perfectly balanced)

### 예 (t=3)
```
      [10, 20, 30]
     /  /  \  \
  [<10][10-20][20-30][>30]
```

---

## 4. Search

```python
def search(node, key):
    i = 0
    while i < len(node.keys) and key > node.keys[i]:
        i += 1
    if i < len(node.keys) and key == node.keys[i]:
        return (node, i)
    if node.is_leaf:
        return None
    return search(node.children[i], key)
```

### 시간 — O(log_t N)

---

## 5. Insert

### 단계
1. Leaf 까지 찾아 내려가기
2. Leaf 에 추가
3. 노드 가득 (2t-1 키) — split

### Split
```
한 노드 [a, b, c, d, e, f, g]  (2t-1=7, t=4)
         ↓ split at middle
부모에 d 올림:
       [..., d, ...]
       /      \
   [a,b,c]  [e,f,g]
```

### 시간 — O(t × log_t N)

---

## 6. Delete

### 복잡 — 3 case
1. Leaf 의 키 — 직접 제거 (if size > t-1)
2. Internal node 의 키 — predecessor 또는 successor 와 swap → 삭제
3. Underflow — 형제로부터 빌리거나 merge

---

## 7. B+Tree

### 차이
- **모든 값 — leaf 에만** (internal 은 index 만)
- Leaf 들 — linked list (순차 접근)

```
     [10, 20, 30]               <- internal (index only)
    /   /   \   \
[10*][20*][30*][40*]           <- leaf (값, 다음 leaf 가리킴)
  →    →    →    →
```

### 효과
- Range query — leaf chain 따라가기
- 모든 키 — 같은 깊이 (검색 일관)

### 사용
- 거의 모든 DB index (MySQL InnoDB, PostgreSQL, Oracle)
- 파일 시스템 (NTFS, ext4, btrfs)

---

## 8. B+ vs B 비교

| | B-Tree | B+Tree |
| --- | --- | --- |
| **Values** | 모든 노드 | Leaf 만 |
| **Range query** | In-order traversal | Leaf chain |
| **Internal node** | 키 + 값 | 키만 |
| **Branching** | 적음 (값 차지) | 많음 (키만) |
| **사용** | 옛 | 모던 (DB) |

---

## 9. DB 에서

### MySQL InnoDB
- **Clustered index** (primary key) — B+Tree
- Leaf — 실제 row data
- Secondary index — leaf 가 primary key

### PostgreSQL
- B+Tree (default)
- 다양한 type (B-tree, hash, GiST, GIN, ...)

### Oracle
- B+Tree

### MongoDB
- B-tree (조금 옛 표현, 실제 B+Tree-like)

### Lucene (Elasticsearch)
- Inverted index — 다른 구조

자세히 → database/index

---

## 10. Concurrency — B-tree

### Latching
- 노드 별 lock
- 위에서 아래로 — top-down

### Lock coupling
- 부모 lock → 자식 lock → 부모 unlock

### B-link tree
- 형제 포인터 — split 시 일관성

자세히 → database/concurrency

---

## 11. 파일 시스템 — B-tree

### ext4
- Htree (B-tree variant)
- 디렉토리 — fast lookup

### NTFS
- B+Tree (directory)

### Btrfs
- B-tree 기반 전체

### ReiserFS
- B+Tree

### APFS (Apple)
- B-tree

---

## 12. 변형

### B*-Tree
- Split 시 — 2 → 3 (더 차게)
- 메모리 효율

### Bε-Tree
- Write 빠름 (LSM 의 부드러운 변형)
- TokuDB

### Copy-on-Write B-Tree
- Persistent / immutable
- 자세히 → DDIA / Btrfs

### Fractal Tree
- Tokutek (현 Percona)

---

## 13. 시간 / 공간

### 시간
- Search / Insert / Delete — O(log_t N)
- t 클수록 — 트리 얕음 → disk read ↓

### 공간
- O(N) 키
- Disk page utilization — 50-100% (split 후 50%)

---

## 14. Page Size 선택

| Page Size | t (대략) | 사용 |
| --- | --- | --- |
| 4 KB | 100+ | 일반 |
| 8 KB | 200+ | DB |
| 16 KB | 400+ | InnoDB |
| 32 KB | 800+ | 큰 record |

### 트레이드오프
- 큰 page — branching ↑, 트리 얕음
- 작은 page — disk read 적음 (메모리 압박 적음)

---

## 15. LSM-Tree vs B-Tree

| | B-Tree | LSM-Tree |
| --- | --- | --- |
| **Read** | 빠름 (한 트리) | 느림 (여러 SST 검색) |
| **Write** | 느림 (in-place) | 빠름 (append-only) |
| **Compaction** | 없음 | 필요 |
| **Space amp** | 작음 | 큼 |
| **사용** | RDBMS | RocksDB / Cassandra / LevelDB |

자세히 → [[../advanced/lsm-tree]] (TBD)

---

## 16. 함정

### 함정 1 — Page utilization
Split 후 — 50% 차 있는 노드. Compaction 필요.

### 함정 2 — Random insert
큰 PK insert — last page 채우기. 좋음.
Random PK (UUID) — 여러 page write — fragmentation.
→ UUID v7 / TSID 권장.

### 함정 3 — Update 의 amplification
한 row 변경 → 전체 page write (16 KB).

### 함정 4 — Long key
키 너무 크면 — branching ↓. Prefix compression.

### 함정 5 — Concurrent latching
Naive — 데드락. Lock coupling / B-link.

### 함정 6 — Tree height 의 증가
N 폭증 시 — height +1 = disk read +1.

---

## 17. 학습 자료

- CLRS Ch 18
- "Database Internals" (Petrov)
- "The Ubiquitous B-Tree" — Comer (1979)
- DDIA Ch 3

---

## 18. 관련

- [[trees]] — Hub
- [[bst]] / [[avl-tree]] / [[red-black-tree]]
- [[../advanced/lsm-tree]] (TBD) — 비교
- [[../../database-theory/database-theory]] — DB index
