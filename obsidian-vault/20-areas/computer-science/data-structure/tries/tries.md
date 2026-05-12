---
title: "트라이 (Tries)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T09:00:00+09:00
tags:
  - data-structure
  - trie
  - prefix-tree
---

# 트라이 (Tries) — Prefix Tree

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 수준 |

**[[../data-structure|↑ data-structure]]** · **[[../../computer-science|↑↑ computer-science]]**

---

## 1. 한 줄 정의

**문자열을 트리 구조로 저장** 하는 자료구조. 공통 prefix 가 공유 — 자동완성,
사전 검색, IP 라우팅의 토대.

---

## 2. 개념의 깊이

### 2.1 왜 트라이인가

문자열 N 개의 집합에서 "단어가 있나?" 또는 "이 prefix 로 시작하는 단어들?":
- 해시 테이블: 단어 존재만 O(L). prefix 검색 O(N).
- BST/정렬 배열: 단어 O(L log N). prefix 검색 lower_bound + 순회.
- **Trie**: **단어 / prefix 모두 O(L)**.

### 2.2 구조

```
"car", "cat", "card", "care", "dog" 저장:

           root
          /    \
         c      d
         |      |
         a      o
        /|      |
       r t      g*
      /||\
     *d* e*
        
* = 단어 끝 (is_end_of_word)
```

각 노드 = 한 문자 + 자식 포인터 + 끝 표시.

### 2.3 메모리 트레이드오프

각 노드의 자식 표현 3 가지:

| 표현 | 메모리 | 검색 |
| --- | --- | --- |
| **고정 배열 (26 영문)** | O(26 × N) — 큰 낭비 | O(1) |
| **HashMap** | O(degree) | O(1) 평균 |
| **연결 리스트** | O(degree) | O(degree) |

영어 단어 10만 개 → 약 50만 노드. 각 노드 26 포인터면 메모리 폭발. **HashMap**
가장 흔함.

### 2.4 압축 트라이 (Radix Tree / PATRICIA Trie)

분기 없는 노드 사슬을 압축:
```
"car" "card" "care" → "car" + [d, e]

압축:
  ca → r → [d, e]
  (한 노드에 여러 글자)
```

Linux 커널 라우팅 테이블 (IP routing) 이 **radix tree** 사용.

### 2.5 DAFSA (Directed Acyclic Word Graph)

같은 suffix 공유 → 메모리 최소. 단점: 동적 갱신 어려움.

### 2.6 Suffix Tree / Suffix Array

문자열 매칭 / 게놈 분석. Ukkonen 알고리즘 O(N) 구축. 더 깊은 주제.

---

## 3. 복잡도

| 연산 | 시간 | 공간 |
| --- | --- | --- |
| 삽입 (단어 길이 L) | O(L) | O(L) per word |
| 검색 (단어) | O(L) | O(1) |
| 검색 (prefix) | O(L) | O(1) |
| Prefix 로 시작하는 모든 단어 | O(L + 결과 합) | — |
| 전체 메모리 | O(N × L × alphabet) 최악 | — |

---

## 4. 언어별 구현

### Python
```python
class TrieNode:
    def __init__(self):
        self.children = {}
        self.is_end = False

class Trie:
    def __init__(self):
        self.root = TrieNode()
    
    def insert(self, word):
        node = self.root
        for c in word:
            if c not in node.children:
                node.children[c] = TrieNode()
            node = node.children[c]
        node.is_end = True
    
    def search(self, word):
        node = self._find(word)
        return node is not None and node.is_end
    
    def starts_with(self, prefix):
        return self._find(prefix) is not None
    
    def _find(self, s):
        node = self.root
        for c in s:
            if c not in node.children: return None
            node = node.children[c]
        return node

t = Trie()
t.insert("car"); t.insert("cat"); t.insert("card")
t.search("car")          # True
t.search("ca")           # False
t.starts_with("ca")      # True
```

### Java
```java
class Trie {
    static class Node {
        Node[] children = new Node[26];   // 영문 소문자
        boolean isEnd;
    }
    Node root = new Node();
    
    public void insert(String word) {
        Node cur = root;
        for (char c : word.toCharArray()) {
            int i = c - 'a';
            if (cur.children[i] == null) cur.children[i] = new Node();
            cur = cur.children[i];
        }
        cur.isEnd = true;
    }
    
    public boolean search(String word) {
        Node node = find(word);
        return node != null && node.isEnd;
    }
    
    public boolean startsWith(String prefix) {
        return find(prefix) != null;
    }
    
    private Node find(String s) {
        Node cur = root;
        for (char c : s.toCharArray()) {
            int i = c - 'a';
            if (cur.children[i] == null) return null;
            cur = cur.children[i];
        }
        return cur;
    }
}
```

### C++
```cpp
struct TrieNode {
    unordered_map<char, TrieNode*> children;
    bool isEnd = false;
};
class Trie {
    TrieNode *root = new TrieNode();
public:
    void insert(string word) {
        TrieNode *cur = root;
        for (char c : word) {
            if (!cur->children.count(c)) cur->children[c] = new TrieNode();
            cur = cur->children[c];
        }
        cur->isEnd = true;
    }
};
```

### JavaScript
```javascript
class Trie {
    constructor() { this.root = {}; }
    insert(word) {
        let cur = this.root;
        for (const c of word) {
            if (!(c in cur)) cur[c] = {};
            cur = cur[c];
        }
        cur._end = true;
    }
    search(word) {
        const node = this._find(word);
        return node !== null && node._end === true;
    }
    startsWith(prefix) { return this._find(prefix) !== null; }
    _find(s) {
        let cur = this.root;
        for (const c of s) {
            if (!(c in cur)) return null;
            cur = cur[c];
        }
        return cur;
    }
}
```

### Rust (간략)
```rust
use std::collections::HashMap;
struct TrieNode {
    children: HashMap<char, TrieNode>,
    is_end: bool,
}
```

(나머지 언어는 같은 패턴.)

---

## 5. 변형 / 응용

### (a) 자동완성 (Autocomplete)
prefix 검색 후 그 노드 아래 모든 단어 수집 (DFS).

### (b) Spell Checker
편집 거리 + 트라이 → 가까운 단어 찾기.

### (c) IP 라우팅 / Longest Prefix Matching
IPv4 192.168.1.0/24 → 비트 단위 트라이.

### (d) Aho-Corasick — 다중 패턴 매칭
여러 패턴을 한 번에 검색. 트라이 + 실패 함수.

### (e) DNS 조회
도메인 . 으로 끊어 트라이 검색.

### (f) Phonetic / 오타 허용
soundex + 트라이.

### (g) Compressed Trie (Radix Tree)
경로 압축으로 메모리 절약.

---

## 6. 함정 / 안티패턴

### 함정 1 — 메모리 폭발
N=10^6 단어 × 평균 8 글자 × 26 포인터 = 약 200M 노드. HashMap 사용.

### 함정 2 — 영문만 가정
한글 / 유니코드 / 이모지 등 → HashMap 자식 또는 코드포인트.

### 함정 3 — root 노드의 글자
보통 root 는 빈 노드. 첫 글자는 root 의 자식.

### 함정 4 — 단어 끝 표시 누락
"car" 와 "card" 모두 저장 시 "car" 의 r 노드에 `is_end=True` 필요.

### 함정 5 — 삭제의 복잡성
다른 단어가 prefix 공유 → 노드 즉시 삭제 X. 자식 모두 없을 때만.

---

## 7. 대표 문제

### Q1. Implement Trie (LeetCode 208)
위 코드 참조.

### Q2. Word Search II (LeetCode 212)
격자에서 단어 사전 모두 찾기. **트라이 + DFS** 가 표준.

### Q3. Replace Words (LeetCode 648)
루트 사전으로 단어 치환.

### Q4. Maximum XOR (LeetCode 421)
이진 트라이 — 비트 단위 트리.

---

## 8. 연습 문제

- [LeetCode 208 Implement Trie](https://leetcode.com/problems/implement-trie-prefix-tree/)
- [LeetCode 212 Word Search II](https://leetcode.com/problems/word-search-ii/)
- [LeetCode 648 Replace Words](https://leetcode.com/problems/replace-words/)
- [LeetCode 421 Max XOR](https://leetcode.com/problems/maximum-xor-of-two-numbers-in-an-array/)
- [백준 14425 문자열 집합](https://www.acmicpc.net/problem/14425)

---

## 9. 학습 자료

- CLRS Ch.11 (해시 + 트라이)
- "이코테" 부록 — 기타 알고리즘
- [VisuAlgo — Trie](https://visualgo.net/en/recursion)

---

## 10. 관련

- [[../trees/trees]] — 트라이는 트리의 변형
- [[../hash-tables/hash-tables]] — 자식 표현
- [[../../algorithm/algorithm|↑ algorithm]]
- [[../data-structure|↑ data-structure]]
