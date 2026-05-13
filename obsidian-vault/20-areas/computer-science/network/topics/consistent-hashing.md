---
title: "Consistent Hashing"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T10:35:00+09:00
tags:
  - network
  - distributed-systems
  - load-balancing
---

# Consistent Hashing

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Ring / Maglev / Rendezvous |

**[[topics|↑ 토픽 hub]]** · **[[../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

분산 시스템에서 **노드 변경 시 영향 최소** 화하는 해시 알고리즘. Cache / LB / DB 의 핵심.

---

## 2. 왜?

### 단순 해시
```
def lookup(key, n):
    return hash(key) % n
```

- N 변경 (node 추가/제거) → 모든 key 의 노드 변경
- Cache 거의 무효

### Consistent Hashing
- N 변경 → 약 1/N 만 영향
- 큰 시스템 친화

---

## 3. Karger 1997 — 원조

### 논문
"Consistent Hashing and Random Trees" — Karger, Lehman, Leighton, Levine, Lewin, Panigrahy

### Akamai 의 기반
- CDN 서버 분산
- 한 객체 — 같은 server 에 cache

---

## 4. Ring 알고리즘

### 구성
```
Hash space — 0 ~ 2^32 - 1 (원)

서버 hash → ring 위
키 hash → ring 위 — 시계방향 다음 서버

ring:
  ┌─ S1 (hash 1000) ────────────────────┐
  │                                      │
  │                                      │
  │              K1 (1500) ──> S2        │
  │                                      │
  │                                      │
  │   S3 (3000)                          │
  │                                      │
  └─ S2 (2000) ─────────────────────────┘
```

### Lookup
```
def lookup(key):
    h = hash(key)
    return next_server_clockwise(h)
```

### 노드 추가 / 제거
- 노드 A 제거 → A 가 담당하던 키만 next 노드 (B) 로
- 나머지 — 영향 X

---

## 5. Virtual Nodes

### 문제
- 노드 개수 적으면 — 불균등 분산
- 한 노드 — 큰 범위 담당

### 해결
- 각 server 의 **여러 hash** (e.g., 100 virtual)
- Ring 위에 골고루

### 효과
- 균등 분산
- 노드 추가 시 — 영향 골고루

---

## 6. Maglev Hashing (Google, 2016)

### 정의
- Consistent hashing 의 변형
- **Lookup table** 미리 만듦 — O(1)

### 동작
```
1. Permutation table (각 backend 별 순서)
2. Lookup table — 모든 slot (예: 65537) 에 backend
3. key hash → slot → backend
```

### 효과
- O(1) lookup
- 매우 균등 (1% 이내)
- 분산 LB — 같은 결정 (일관성)

### 사용
- Google Maglev
- Envoy

---

## 7. Rendezvous Hashing (HRW)

### 정의
- Highest Random Weight
- 모든 backend 에 hash — 최고 score 의 backend

```python
def lookup(key, backends):
    return max(backends, key=lambda b: hash(key + b.id))
```

### 효과
- Virtual node 불필요
- 노드 추가 — 1/N 만 영향
- 매우 균등

### 사용
- Memcached client (Citrix)
- 일부 분산 cache

### Maglev 와 비교
- Rendezvous — 각 lookup O(N)
- Maglev — table 만들면 O(1)
- 매우 큰 N — Maglev 빠름

---

## 8. Jump Hash (Google, 2014)

### 정의
- "A Fast, Minimal Memory, Consistent Hash Algorithm"
- Lamping & Veach

### 동작
```python
def jump_hash(key, n):
    b, j = -1, 0
    while j < n:
        b = j
        key = (key * 2862933555777941757 + 1) & 0xFFFFFFFFFFFFFFFF
        j = int((b + 1) * (1 << 31) / ((key >> 33) + 1))
    return b
```

### 효과
- Memory 없음 (O(1) space)
- 매우 빠름
- 균등

### 한계
- Backend 순서 — 추가만 OK, 제거 시 — 다음 노드로
- 임의 제거 X — Rendezvous 권장

---

## 9. 사용 — Distributed Cache

### Memcached
- 클라이언트 측 consistent hashing (Ketama)
- 한 server 죽으면 — 1/N 만 cache miss

### Redis Cluster
- 슬롯 (16384 개) 기반
- 비슷 — 노드 추가 시 슬롯 마이그레이션

---

## 10. 사용 — Sharded DB

### DynamoDB
- Partition key — hash → partition (consistent hash 의 변형)

### Cassandra
- Token ring (Murmur3) — virtual nodes (vnodes)

### Riak
- Pure Karger ring

---

## 11. 사용 — LB

### HAProxy
```
backend cache_servers
    balance hash $request_header(X-Cache-Key)
    hash-type consistent
    server c1 ...
    server c2 ...
```

### Nginx
```
upstream cache {
    hash $request_uri consistent;
    server c1;
    server c2;
}
```

### Envoy
- Maglev / Ring hash 옵션

### Cloudflare
- L4 LB — Maglev

---

## 12. Hot key / Hot spot 문제

### 문제
- 한 key 가 매우 hot — 한 backend overload
- "Justin Bieber problem"

### 해결
- Replication — 인기 key 여러 backend
- Power of Two — 2 곳 시도
- Front-line cache (Memcached → app cache)

---

## 13. Resharding 의 도전

### Consistent hashing 도 — 데이터 마이그레이션 필요
```
노드 추가 → 일부 key 가 새 노드로
→ 데이터 이동 필요
```

### 도구
- Background migration
- Tombstone (옛 location 가리키기)
- Eventual consistency

---

## 14. Stable Replicas

### 정의
- 각 key — 여러 backend (3 replica)
- 첫 살아있는 backend 사용

### 효과
- HA / failover
- Read replica

---

## 15. CRUSH (Ceph)

### 정의
- 계층 + consistent hashing
- 토폴로지 (rack / host) 기반 분산
- Failure domain 인식

### 사용
- Ceph storage

---

## 16. 함정

### 함정 1 — Virtual node 너무 적음
20 노드 × 10 vnode = 200 slot — 불균등.
권장 100-200 vnode/node.

### 함정 2 — Hash 함수 약함
MD5 / FNV / Murmur — 다양. SHA1 / Murmur3 권장.

### 함정 3 — 동적 변경의 인식 차이
LB 마다 노드 목록 다름 → 다른 결정. Coordination 필요 (etcd / ZK).

### 함정 4 — Backend 의 weight
같은 spec 가정. 다른 spec — weighted consistent hash.

### 함정 5 — Hot key 의 hot spot
1 key — 1 backend. Replicated hot key + cache.

### 함정 6 — Stateful migration
노드 추가 시 — 데이터 이동 비용. Cassandra / Redis Cluster — 운영 도전.

---

## 17. 학습 자료

- "Consistent Hashing and Random Trees" (Karger 1997)
- "Maglev" (Google 2016)
- "Jump Hash" (Lamping 2014)
- "DDIA" (Kleppmann) — partitioning

---

## 18. 관련

- [[topics]] — Hub
- [[../load-balancing/load-balancing-algorithms]] — LB
- [[../dns/dns]] — Akamai DNS 의 활용
