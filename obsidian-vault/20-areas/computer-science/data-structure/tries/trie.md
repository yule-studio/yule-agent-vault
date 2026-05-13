---
title: "Trie — Prefix Tree"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T17:00:00+09:00
tags:
  - data-structure
  - trie
---

# Trie — Prefix Tree

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 구조 / 연산 / 응용 |

**[[tries|↑ Tries]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 한 줄 정의

**문자열의 prefix 를 공유하는 트리**. 각 노드 — 한 문자. Insert / Search — O(K) (K = 문자열 길이).

---

## 2. 구조

```
"apple", "app", "apt", "bat"

         root
        / | \
       a  b
      /   |
     p    a
    /|    |
   p t    t
   |
   l
   |
   e
```

### 노드
```python
class TrieNode:
    def __init__(self):
        self.children = {}    # char → node
        self.is_end = False
        self.count = 0         # 변형: 단어 수
```

---

## 3. 연산

### Insert
```python
def insert(root, word):
    cur = root
    for c in word:
        if c not in cur.children:
            cur.children[c] = TrieNode()
        cur = cur.children[c]
    cur.is_end = True
```

### Search
```python
def search(root, word):
    cur = root
    for c in word:
        if c not in cur.children:
            return False
        cur = cur.children[c]
    return cur.is_end
```

### Starts with prefix
```python
def starts_with(root, prefix):
    cur = root
    for c in prefix:
        if c not in cur.children:
            return False
        cur = cur.children[c]
    return True
```

### Delete
```python
def delete(node, word, depth=0):
    if not node: return False
    if depth == len(word):
        if not node.is_end: return False
        node.is_end = False
        return len(node.children) == 0
    c = word[depth]
    if delete(node.children.get(c), word, depth + 1):
        del node.children[c]
        return not node.is_end and len(node.children) == 0
    return False
```

---

## 4. 시간 / 공간

| | 시간 |
| --- | --- |
| Insert | O(K) (K = word length) |
| Search | O(K) |
| Starts with | O(K) |
| Delete | O(K) |

| 공간 | O(N × K) (N = words, K = avg length) |

### Hash table 과 비교
- Hash — Search O(K) (hash 계산)
- Trie — Search O(K) (prefix sharing)
- Trie — Prefix query 가능 (Hash X)

---

## 5. 구현 — 배열 vs Hash

### Children — Fixed array
```python
class TrieNode:
    def __init__(self):
        self.children = [None] * 26    # for 'a'-'z'
        self.is_end = False
```

#### 장점
- O(1) child access
- Cache 친화

#### 단점
- 메모리 ↑ (대부분 None)
- Sparse alphabet — 큰 낭비

### Children — Hash map
```python
class TrieNode:
    def __init__(self):
        self.children = {}
        self.is_end = False
```

#### 장점
- 메모리 효율
- 모든 char 지원

#### 단점
- Hash overhead

---

## 6. 응용

### 6.1 자동완성 (Autocomplete)
```python
def autocomplete(root, prefix):
    cur = root
    for c in prefix:
        if c not in cur.children:
            return []
        cur = cur.children[c]
    
    results = []
    def dfs(node, path):
        if node.is_end:
            results.append(prefix + path)
        for c, child in node.children.items():
            dfs(child, path + c)
    dfs(cur, "")
    return results
```

### 6.2 사전 검사 (Word break)
- "applepie" → ["apple", "pie"]
- Trie 로 빠른 prefix 매칭

### 6.3 IP 라우팅 (Longest prefix match)
- 라우터 — 자세히 [[../advanced/radix-tree]] (TBD)
- 1.2.3.0/24 / 1.2.3.0/28 등

### 6.4 Spell check
- Dictionary 의 모든 단어 trie
- Edit distance 와 결합

### 6.5 Pattern matching (Aho-Corasick)
- 여러 pattern 의 동시 매칭
- Trie + failure links

### 6.6 Bit Trie (XOR maximization)
```python
# Max XOR — 정수의 bit trie
def max_xor(root, num):
    cur = root
    result = 0
    for i in range(31, -1, -1):
        bit = (num >> i) & 1
        opposite = 1 - bit
        if opposite in cur.children:
            result |= (1 << i)
            cur = cur.children[opposite]
        else:
            cur = cur.children[bit]
    return result
```

---

## 7. 변형 — Compressed / Radix / Patricia

### 정의
- 한 노드 — 여러 char (chain 압축)
- 메모리 절약

자세히 → [[radix-tree]]

---

## 8. Suffix Trie / Suffix Tree

### Suffix Trie
- 모든 suffix 의 trie
- O(N²) 공간 (naive)

### Suffix Tree (Ukkonen, 1995)
- 압축
- O(N) 공간 + O(N) build

### 사용
- 문자열 검색
- LCP (Longest Common Substring)
- Bioinformatics (DNA)

자세히 → algorithm-advanced

---

## 9. Trie + DP

### Word Break II
- 문자열 → 단어들의 시퀀스 (가능한 모든 분리)
- Trie + recursion + memo

### Concatenated Words
- 다른 단어들의 결합인지

---

## 10. Java / Python — 표준 없음

### Java
- 직접 구현
- Apache Commons Collections 의 PatriciaTrie

### Python
- 직접 또는 `pygtrie` 라이브러리

### Go
- `github.com/tchap/go-patricia`

---

## 11. 메모리 — 큰 trie

### Naive
- N = 10^6 단어, K = 10
- 노드 = 10^7
- 각 노드 — children 26 (array) → 10^7 × 26 × 8 byte = 2 GB

### 압축
- Radix tree — 노드 수 ↓
- DAWG (Directed Acyclic Word Graph) — 공통 suffix 도 공유
- Marisa Trie — 매우 효율 (NLP)

---

## 12. 함정

### 함정 1 — 메모리 폭증
ASCII (256) 모든 위치 — 큰 낭비. Hash / radix.

### 함정 2 — Delete 의 cascading
한 단어 삭제 — 노드 chain 제거 (다른 단어 사용 안 하면).

### 함정 3 — Case sensitivity
대소문자 — 별도 node. Normalize.

### 함정 4 — Unicode
한 char ≠ 한 byte. Code point / grapheme.

### 함정 5 — Persistence
큰 trie — disk 저장 어려움. Serialize / Marisa.

### 함정 6 — Iteration 순서
대부분 — 알파벳 순.

---

## 13. 학습 자료

- CLRS — Trie (간단)
- "Algorithms" Sedgewick — String chapter
- LeetCode Trie tag
- "Suffix Trees and Suffix Arrays" — Gusfield

---

## 14. 관련

- [[tries]] — Hub
- [[radix-tree]] — 압축
- [[trie-applications]] — 사용 사례
- [[../../algorithm/algorithm]] — string matching
