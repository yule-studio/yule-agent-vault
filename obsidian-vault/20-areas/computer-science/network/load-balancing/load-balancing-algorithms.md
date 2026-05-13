---
title: "Load Balancing 알고리즘"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T06:15:00+09:00
tags:
  - network
  - load-balancing
  - algorithms
  - consistent-hash
---

# Load Balancing 알고리즘

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | RR / Least Conn / Consistent Hash / P2C |

**[[load-balancing|↑ Load Balancing]]** · **[[../network|↑↑ network hub]]**

---

## 1. 알고리즘 — 한눈

| 알고리즘 | 특징 |
| --- | --- |
| **Round-Robin** | 순서대로 분배 |
| **Weighted Round-Robin** | 가중치 따라 비율 |
| **Random** | 랜덤 |
| **Least Connections** | 활성 연결 수 적은 곳 |
| **Least Response Time** | 평균 응답 시간 짧은 곳 |
| **IP Hash** | 클라이언트 IP hash |
| **Consistent Hash** | 키 hash (백엔드 변경 시 안정) |
| **EWMA** | Exponentially Weighted Moving Average |
| **Power of Two Choices** | 2 개 랜덤 → 더 좋은 곳 |

---

## 2. Round-Robin

```
요청 1 → Server A
요청 2 → Server B
요청 3 → Server C
요청 4 → Server A   (다시)
```

### 장점
- 단순 / 빠름
- 균등 분산 (요청 수)

### 단점
- 요청별 비용 다르면 불균등
- Server capacity 차이 무시

---

## 3. Weighted Round-Robin

```
A (weight 3), B (weight 1), C (weight 1)

순서: A A A B C A A A B C ...
```

### 사용
- 다른 spec 의 서버 (capacity 차이)
- Canary deployment

---

## 4. Least Connections

```
A: 5 연결
B: 3 연결    ← 선택
C: 7 연결
```

### 장점
- 부하 차이 보상
- Long-lived (WebSocket) 친화

### 단점
- 연결 수 ≠ 처리 비용
- 카운터 동기 (분산 LB)

---

## 5. Weighted Least Connections

```
A (capacity 10, 5 conn → ratio 0.5)
B (capacity 5, 3 conn → ratio 0.6)
C (capacity 10, 3 conn → ratio 0.3)  ← 선택
```

→ Capacity 보정.

---

## 6. Least Response Time

```
A: avg 100 ms
B: avg 50 ms    ← 선택
C: avg 150 ms
```

### 장점
- 실측 성능 반영
- Slow backend 회피

### 단점
- 측정 / 캐시
- 분산 LB 의 일관성

---

## 7. IP Hash

```
hash(client_ip) % N → backend
```

### 효과
- 같은 클라이언트 — 같은 backend (sticky)
- Session 친화 (cookie 없이)

### 단점
- N 변경 시 — 모두 다른 backend (cache miss)
- NAT 뒤 — 한 IP 의 많은 클라이언트 — 한 backend

---

## 8. Consistent Hashing

### 핵심 아이디어
- Hash ring 위에 backend 와 key 배치
- Key 의 hash 가 가까운 backend
- Backend 추가 / 제거 — 영향 최소 (1/N 만)

### 알고리즘
```
ring = sorted([hash(b1), hash(b2), ..., hash(bN)])

def lookup(key):
    h = hash(key)
    return next_backend_on_ring(h)
```

### Virtual nodes
- 각 backend 의 여러 hash (예: 100 개 virtual)
- 균등 분산 개선

### 사용
- Memcached (Ketama)
- DynamoDB / Cassandra / Riak
- HAProxy `hash` directive
- Akamai / Cloudflare CDN

자세히 → [[../topics/consistent-hashing]] (TBD)

---

## 9. Maglev Hashing (Google)

### 정의
- Consistent hashing 의 변형
- 모든 backend → 큰 lookup table (전치)
- O(1) lookup

### 효과
- 매우 균등
- 분산 LB 의 일관성

### 논문
- "Maglev" (2016)

---

## 10. Rendezvous (Highest Random Weight)

```
def lookup(key):
    return argmax(b in backends) of hash(key, b)
```

### 효과
- Consistent
- Virtual node 불필요

### 사용
- 분산 cache (Citrix Memcached)
- Brave 의 일부

---

## 11. Power of Two Choices (P2C)

```
def lookup():
    a = random_backend()
    b = random_backend()
    return min(a, b) by load
```

### 효과
- 거의 best 와 같이 균등 (이론)
- 분산 친화 (각 LB 독립)

### 사용
- HAProxy `random` + check
- NGINX Plus

### 논문
- "The Power of Two Choices in Randomized Load Balancing" (Mitzenmacher)

---

## 12. EWMA — Exponentially Weighted Moving Average

```
avg_latency = α * new + (1-α) * old
```

### 효과
- 최근 측정 가중
- Slow backend 빠른 회피

### 사용
- Envoy `LEAST_REQUEST` 변형
- gRPC 의 일부

---

## 13. Random

```
return random_backend()
```

### 장점
- 매우 단순
- 분산 LB 에서 일관성 좋음

### 단점
- 균등성 — 큰 N 에서만

---

## 14. 적용 — 시나리오 별

| 시나리오 | 알고리즘 |
| --- | --- |
| 일반 HTTP | RR / Random |
| Long-lived (WebSocket) | Least Conn |
| Cache backend | Consistent Hash |
| 다른 capacity | Weighted |
| Latency 차이 큼 | Least RT / EWMA |
| Service Mesh | P2C / Random + EWMA |

---

## 15. 분산 LB 의 일관성

### 문제
- 여러 LB 가 같은 backend pool → 카운트 / latency 동기?

### Solution
- **Random / Hash** — 동기 불필요
- **Least Conn** — 자기 카운트만 + best-effort
- **P2C** — 분산 친화

### Maglev / Katran
- 모든 LB 가 같은 결정 (consistent hashing)

---

## 16. 함정

### 함정 1 — Round-Robin 의 hot key
큰 요청 1 개 → 한 backend overload.
Least Conn / Least RT 권장.

### 함정 2 — Consistent Hash 의 hot spot
한 키가 매우 hot → 한 backend.
대안 — request shedding / additional replication.

### 함정 3 — IP Hash 의 NAT
한 IP 의 많은 사용자 — 한 backend.

### 함정 4 — Least Conn 의 분산 카운트 부정확
여러 LB → 자기 카운트만. Best-effort.

### 함정 5 — P2C 의 측정 변동
짧은 측정 — noise. EWMA 결합.

### 함정 6 — Sticky session 의 scale-down
한 backend 죽으면 session 분산.

---

## 17. 학습 자료

- "Consistent Hashing and Random Trees" (Karger 1997)
- "Maglev" Google (2016)
- "Power of Two Choices" (Mitzenmacher)
- HAProxy / NGINX / Envoy LB docs

---

## 18. 관련

- [[load-balancing]] — Hub
- [[l4-load-balancing]] — Maglev / Katran
- [[l7-load-balancing]] — Nginx / Envoy
- [[health-check]] — Backend 선택의 전제
