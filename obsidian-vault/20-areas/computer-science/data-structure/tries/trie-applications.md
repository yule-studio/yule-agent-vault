---
title: "Trie 응용 사례"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T17:10:00+09:00
tags:
  - data-structure
  - trie
  - applications
---

# Trie 응용 사례

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | autocomplete / NLP / 라우팅 |

**[[tries|↑ Tries]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. 응용 — 한눈

| 분야 | 사용 |
| --- | --- |
| **Autocomplete** | 검색 / IDE / 입력기 |
| **Spell check** | 사전 / browser |
| **IP routing** | 라우터 의 LPM |
| **URL routing** | 웹 프레임워크 |
| **Game (Scrabble)** | 단어 매칭 |
| **Bioinformatics** | DNA / 단백질 |
| **Compiler** | Keyword / symbol table |
| **Aho-Corasick** | 여러 pattern 동시 매칭 |

---

## 2. Autocomplete

### Google / 검색
- 사용자 입력 — prefix
- Trie 에서 candidates 검색
- Frequency / personalization 로 ranking

### IDE
- 변수 / 함수 이름
- LSP — trie 기반 fuzzy matching

### 구현
```python
class Autocomplete:
    def __init__(self):
        self.root = TrieNode()
    
    def insert(self, word, freq):
        cur = self.root
        for c in word:
            if c not in cur.children:
                cur.children[c] = TrieNode()
            cur = cur.children[c]
        cur.is_end = True
        cur.freq = freq
    
    def suggest(self, prefix, k=10):
        cur = self.root
        for c in prefix:
            if c not in cur.children: return []
            cur = cur.children[c]
        
        # DFS — collect all words under cur
        suggestions = []
        def dfs(node, path):
            if node.is_end:
                suggestions.append((node.freq, prefix + path))
            for c, child in node.children.items():
                dfs(child, path + c)
        dfs(cur, "")
        suggestions.sort(reverse=True)
        return [w for _, w in suggestions[:k]]
```

---

## 3. Spell check

### 정의
- Dictionary 의 trie + edit distance
- 입력 → 가까운 단어 추천

### 알고리즘
- BK Tree (Burkhard-Keller)
- Levenshtein automaton (Schulz, 2002)
- Trie + Dynamic Programming

### Norvig's spell corrector
- 1-edit 후보 — 모두 생성
- Dictionary 에 있는 것 매치 (trie / set)

---

## 4. IP Routing — Longest Prefix Match

### 라우터의 핵심
- 도착 IP — 가장 긴 매치 prefix 의 next-hop

### Trie / Radix
- 32-bit IPv4 — bit 별 trie
- Compressed (level-compressed)

### 모던
- DPDK LPM library
- LC-Trie (Linux kernel)
- Tree bitmap (Eatherton-Varghese)

자세히 → [[radix-tree]] / [[../../network/routing/bgp]]

---

## 5. URL routing (Web framework)

### Go httprouter
```go
router.GET("/users/:id", handler)
router.GET("/users/:id/posts", handler)
```

### Trie 기반
- Path part — trie node
- `:id` — wildcard

### 효과
- O(K) lookup (K = path depth)
- 빠름 (대부분 frameworks)

### 사용
- Gin, Echo, Fiber (Go)
- Express (Node.js)
- FastAPI (Python)

---

## 6. Aho-Corasick — Multi-pattern matching

### 정의
- 여러 pattern 동시 검색 — O(N + M + total matches)
- Trie + failure links (KMP 일반화)

### 사용
- Antivirus (signature)
- Network intrusion detection
- Spam filter
- NLP (named entity)

### 구현
```python
class AhoCorasick:
    def __init__(self):
        self.trie = {}
        self.fail = {}
        self.output = {}
    
    def add(self, pattern, id):
        # ... insert + setup failure links via BFS ...
    
    def search(self, text):
        # ... follow trie + fail links ...
```

---

## 7. Bioinformatics

### DNA / RNA sequence
- ATCG (4 chars)
- Genome search — billions of bases

### 사용
- Suffix tree / array
- FM-index
- BWA / Bowtie (align reads)

---

## 8. Compiler

### Symbol Table
- 변수 / 함수 이름
- 일부 — trie (이름 prefix 검색)
- 대부분 — hash table

### Keyword recognition
- Lexer 의 keyword
- Trie 또는 perfect hash

---

## 9. Word break / Concatenated words

### Word Break
- "applepie" — ["apple", "pie"]
- Trie 로 빠른 prefix check

```python
def word_break(s, wordDict):
    trie = build_trie(wordDict)
    n = len(s)
    dp = [False] * (n + 1)
    dp[0] = True
    for i in range(n):
        if not dp[i]: continue
        cur = trie
        for j in range(i, n):
            if s[j] not in cur.children: break
            cur = cur.children[s[j]]
            if cur.is_end:
                dp[j + 1] = True
    return dp[n]
```

---

## 10. Stream pattern matching

### IDS (Intrusion Detection)
- 네트워크 packet — 패턴 매칭
- Snort / Suricata

### Trie + Aho-Corasick
- 수천 pattern — 한 패스

---

## 11. Markov chain — Text generation

### n-gram → trie
- "the cat sat" → 다음 단어 예측
- Trie — n-gram count

### 사용
- 옛 NLP (현재 — LLM)
- 일부 keyboard autocomplete

---

## 12. Anagram / Word puzzle

### Boggle game
- Board 의 글자 — DFS + trie 사용
- Trie — 사전 단어

### Scrabble
- Rack 의 글자 + board → 가능한 단어
- DAWG (compressed trie)

---

## 13. 함정

### 함정 1 — 메모리
큰 사전 — naïve trie 의 메모리 폭증. Compressed (radix) / DAWG.

### 함정 2 — Disk-resident trie
RAM 부족 — 일부 disk. Marisa / RDX.

### 함정 3 — Concurrent
Lock-free trie — 매우 복잡.

### 함정 4 — Unicode 의 char
Code point ≠ user-visible char. Grapheme.

### 함정 5 — 자주 update
큰 trie — insert/delete 비싸. Cache / batch.

---

## 14. 학습 자료

- "Information Retrieval" — Manning
- "Algorithms" Sedgewick — String
- Norvig's spell corrector blog
- Linux fib_trie source

---

## 15. 관련

- [[tries]] — Hub
- [[trie]] / [[radix-tree]]
- [[../../network/routing/bgp]] — Routing
- [[../../algorithm/algorithm]] — String matching
