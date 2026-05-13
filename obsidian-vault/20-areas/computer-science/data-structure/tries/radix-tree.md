---
title: "Radix Tree / Patricia Tree — 압축 Trie"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T17:05:00+09:00
tags:
  - data-structure
  - trie
  - radix
  - patricia
---

# Radix Tree / Patricia Tree

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 압축 / IP routing / suffix tree |

**[[tries|↑ Tries]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

**Trie 의 한 자식만 가진 chain 노드 — 합쳐서 한 노드** (string 으로). 메모리 효율 + 같은 시간.

---

## 2. Trie → Radix

### Trie
```
"apple", "app"
     a
     |
     p
     |
     p   ← is_end (= "app")
     |
     l
     |
     e   ← is_end (= "apple")
```

### Radix Tree
```
"apple", "app"
     "app"   ← is_end
       |
      "le"   ← is_end
```

---

## 3. 노드

```python
class RadixNode:
    def __init__(self):
        self.children = {}    # first_char → (substring, child_node)
        self.is_end = False
```

---

## 4. Insert / Search

### Insert
```python
def insert(node, word):
    if not word:
        node.is_end = True
        return
    
    first = word[0]
    if first not in node.children:
        # 새 자식
        node.children[first] = (word, RadixNode())
        node.children[first][1].is_end = True
        return
    
    edge, child = node.children[first]
    
    # 공통 prefix 찾기
    i = 0
    while i < len(edge) and i < len(word) and edge[i] == word[i]:
        i += 1
    
    if i == len(edge):
        # edge 가 모두 매치 — child 로 내려감
        insert(child, word[i:])
    elif i == len(word):
        # word 모두 매치 — edge 분할 (앞부분만)
        split = RadixNode()
        split.is_end = True
        split.children[edge[i]] = (edge[i:], child)
        node.children[first] = (word, split)
    else:
        # 분할
        new = RadixNode()
        new.children[edge[i]] = (edge[i:], child)
        new_node = RadixNode()
        new_node.is_end = True
        new.children[word[i]] = (word[i:], new_node)
        node.children[first] = (edge[:i], new)
```

(실제 구현 — 복잡)

---

## 5. Patricia Trie

### Patricia — Practical Algorithm to Retrieve Information Coded in Alphanumeric (Morrison, 1968)

### 정의
- Radix tree 의 binary 변형 (k=2)
- Bit string 의 비교

### 사용
- IP routing
- Crit-bit tree (Bernstein)

---

## 6. 응용 — IP Routing

### Longest Prefix Match
- 라우터 — 도착 IP 에 가장 긴 매치 prefix 의 route 선택

```
IP: 192.168.5.10

Routes:
  192.168.5.0/24  → eth0
  192.168.0.0/16  → eth1
  0.0.0.0/0       → eth2 (default)
```

### Trie 로 LPM
- IP bit 별 — bit trie
- Longest path 가 best match

### 모던 — DPDK / Linux kernel
- Compressed trie + multi-bit (4-bit / 8-bit) — 빠름
- LC-Trie (Level Compressed Trie)

---

## 7. Linux Kernel — fib_trie

### 정의
- Linux IPv4 route lookup
- LC-Trie 기반
- 매우 빠름 (수 µs)

### 변형
- nftables / iptables 의 일부

---

## 8. URL routing (Web framework)

### 정의
- HTTP path → handler 매핑
- "/users/:id/posts/:post_id"

### 구현
- 대부분 — radix tree 또는 trie
- httprouter (Go) / Gin / Express / FastAPI

### 효과
- O(K) lookup
- Wildcard 지원

---

## 9. Suffix Tree

### 정의
- 문자열의 모든 suffix — radix tree

### 시간
- Build — O(N) (Ukkonen 알고리즘)
- 검색 — O(M) (M = pattern length)

### 사용
- 문자열 검색
- LCP / LCS
- Bioinformatics

자세히 → algorithm-string

---

## 10. DAWG — Directed Acyclic Word Graph

### 정의
- Radix tree + 공통 suffix 합치기
- "running", "spring" → "ing" 공유

### 효과
- 매우 효율 (영어 단어 수 → MB)
- 사전 / 게임 (Scrabble)

---

## 11. Marisa Trie

### 정의
- "Matching Algorithm with Recursively Implemented StorAge"
- LOUDS 비트 인코딩
- 매우 작은 메모리

### 사용
- 일본어 IME (Mozc)
- NLP / 검색 엔진

---

## 12. 구현 — 언어별

### Python — `pygtrie`
```python
import pygtrie
t = pygtrie.CharTrie()
t['apple'] = 1
t['app'] = 2

# Prefix lookup
list(t.iteritems('app'))   # [('app', 2), ('apple', 1)]
```

### Go — `github.com/armon/go-radix`
```go
r := radix.New()
r.Insert("apple", 1)
r.Insert("app", 2)
r.Get("app")     // 2, true
```

### Java
- Apache Commons Collections — PatriciaTrie

### Rust — `radix_trie` crate

---

## 13. 함정

### 함정 1 — Edge split
Insert 시 — 공통 prefix 찾고 edge 분할. 코드 복잡.

### 함정 2 — Memory
Edge 의 string — 별도 또는 share.

### 함정 3 — Delete 의 merge
삭제 후 — chain 1 child 만 — merge.

### 함정 4 — 큰 alphabet
Children — hash 또는 sorted list.

### 함정 5 — Concurrent
Lock-free radix — 매우 복잡.

---

## 14. 학습 자료

- Morrison 1968 — Patricia paper
- "Algorithms" Sedgewick — Radix
- Linux fib_trie source
- httprouter source

---

## 15. 관련

- [[tries]] — Hub
- [[trie]] — 기반
- [[trie-applications]]
- [[../../network/routing/bgp]] — Routing
