---
title: "Graph 응용 — 실무 사례"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:30:00+09:00
tags:
  - data-structure
  - graph
  - applications
---

# Graph 응용 — 실무 사례

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | SNS / 도로 / 의존성 / 추천 |

**[[graphs|↑ Graphs]]** · **[[../data-structure|↑↑ data-structure]]**

---

## 1. Graph — 모든 곳

| 분야 | 사례 |
| --- | --- |
| **SNS** | Friend / Follow graph |
| **검색** | Page rank (web graph) |
| **추천** | User-Item graph |
| **도로** | 길찾기 / 교통 |
| **의존성** | Build / Package |
| **AI** | Neural network (DAG) |
| **블록체인** | DAG (IOTA, Hashgraph) |
| **컴파일러** | Control / Data flow |
| **분자** | Atom / bond graph |
| **회로** | 부품 / wire |

---

## 2. SNS — Social Graph

### Facebook / Twitter / LinkedIn
- Friend / Follow — directed edge
- 큰 graph (10^9 노드)

### 도구
- TAO (Facebook) — Graph DB on MySQL
- Neptune (AWS) — graph DB
- Neo4j — popular

### 알고리즘
- 친구 추천 — common friends
- BFS — degrees of separation
- PageRank-like — influence

---

## 3. PageRank — Web Graph

### Larry Page + Sergey Brin (1996)
- Web page — node
- Hyperlink — directed edge

### 알고리즘
```
PR(p) = (1-d)/N + d × Σ PR(t) / L(t)
        ↑ teleport     ↑ 가리키는 페이지 합 (link 수로 정규화)
        d = damping (~0.85)
```

### 효과
- Eigenvalue / Power iteration
- O(iterations × edges)

### 변형
- Personalized PageRank
- Random Walk

---

## 4. 추천 — Bipartite Graph

### User — Item
- Edge — rating / view / purchase

### Collaborative Filtering
- 비슷한 user — 비슷한 item 좋아함
- Cosine similarity

### Graph Neural Network (GNN)
- Embedding — neighborhood aggregation
- Pinterest PinSage / Twitter

---

## 5. 도로 / 교통 — Spatial Graph

### OpenStreetMap (OSM)
- 노드 (좌표) + edge (도로)
- 가중치 — 거리 / 시간 / 톨

### Routing
- Dijkstra / A*
- Bidirectional
- Contraction Hierarchies (OSRM)

### Real-time
- 교통 정체 — edge weight 변경
- Waze / Google Maps

자세히 → [[shortest-path]]

---

## 6. 의존성 — DAG

### Build (Make / Bazel)
- Source — node
- #include — edge

### Package (npm / pip / cargo)
- Package — node
- Dependency — edge
- Topological order — install 순서

### CI / CD pipeline
- Job — node
- Dependency — edge
- DAG scheduling

자세히 → [[topological-sort]]

---

## 7. AI / Neural Network

### Computation Graph
- Operation — node
- Tensor — edge

### 자동 미분 (Autograd)
- Forward — DAG 따라
- Backward — reverse topo

### 도구
- PyTorch (dynamic)
- TensorFlow (static)
- JAX (functional)

---

## 8. 블록체인

### Bitcoin / Ethereum
- Chain (linked list)

### IOTA / Hashgraph (DAG)
- 트랜잭션 — DAG
- 더 빠른 confirmation
- Cycle 없음

### Git
- Commit DAG
- Parent — directed edge

---

## 9. 컴파일러

### Control Flow Graph (CFG)
- Basic block — node
- Branch — edge

### Data Flow Analysis
- Live variable / reaching definition
- Iterative — fix-point

### SSA (Static Single Assignment)
- Variable 한 번만 assign

### Optimization
- Common subexpression / dead code

---

## 10. 분자 / 화학

### Molecular graph
- Atom — node
- Bond — edge

### Drug discovery
- Subgraph matching
- Pattern (functional group)

### Molfile / SMILES
- 분자 인코딩

---

## 11. 회로 — VLSI

### Netlist
- 부품 — node
- Wire — edge

### Place & Route
- Component 배치
- Wire 연결

### Critical Path
- Logic delay
- 최적화

---

## 12. NLP — Knowledge Graph

### Entity — node
- Person / Place / Concept

### Relation — edge
- "born in", "located", "is-a"

### 도구
- DBpedia / Wikidata
- Google Knowledge Graph
- Neo4j Cypher / SPARQL

---

## 13. Graph Databases

### Neo4j
- Cypher query
- Native graph storage
- ACID

### TigerGraph
- 분산
- GSQL

### JanusGraph
- Apache, 분산
- TinkerPop / Gremlin

### Amazon Neptune
- 관리형
- Property graph + RDF

### ArangoDB
- Multi-model
- Graph + document + KV

---

## 14. Graph Processing — 대규모

### Pregel (Google, 2010)
- BSP (Bulk Synchronous Parallel)
- Message passing per super-step

### Apache Giraph
- Pregel 오픈

### GraphX (Spark)
- Spark 위 graph
- 분산 처리

### Gunrock
- GPU 그래프 처리

### Tigergraph / Cosmos DB

---

## 15. Bipartite Matching

### Hungarian algorithm
- Assignment problem
- 작업 → 작업자 배정

### Hopcroft-Karp
- 최대 매칭
- O(E√V)

### Network flow
- 최대 매칭 = 최대 flow
- Ford-Fulkerson / Dinic

자세히 → algorithm-network-flow

---

## 16. Bridge / Articulation Point

### Network reliability
- Bridge 끊기면 — 분리
- Critical infrastructure

### 도시 — choke point

### Tarjan algorithm — DFS based

---

## 17. Bipartite Detection

### 2-coloring
- BFS / DFS

### 사용
- 스포츠 일정 (홈 / 원정)
- 강의 시간표 (충돌)

---

## 18. 함정

### 함정 1 — 큰 graph 의 BFS 메모리
visited — O(V). 비트 array.

### 함정 2 — Disconnected
한 BFS — 한 component. 모든 노드 외부 loop.

### 함정 3 — Cycle 의 의미
Directed — back edge. Undirected — non-parent edge.

### 함정 4 — Multigraph
같은 edge 여러 번 가능 — adj list 검색 시 주의.

### 함정 5 — 자기 자신 가리키기
Self-loop — degree, BFS / DFS 별도 처리.

### 함정 6 — Sparse vs Dense
Matrix vs List 선택 — 사이즈 따라.

---

## 19. 학습 자료

- CLRS Ch 22-26
- "Networks" — Mark Newman
- DDIA Ch 2 — Graph data models
- Neo4j docs

---

## 20. 관련

- [[graphs]] — Hub
- [[graph-representation]] / [[bfs]] / [[dfs]]
- [[shortest-path]] / [[mst]] / [[topological-sort]]
- [[../../network/topics/bgp-incidents]]
