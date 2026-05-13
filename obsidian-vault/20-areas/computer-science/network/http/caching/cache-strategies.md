---
title: "캐싱 전략 (Strategies)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T22:30:00+09:00
tags:
  - network
  - http
  - caching
  - strategy
---

# 캐싱 전략 (Strategies)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Cache-aside / Read-through / Write-through / Write-behind / Refresh-ahead |

**[[caching|↑ Caching]]** · **[[../http|↑↑ HTTP]]**

> HTTP 의 캐싱 (`Cache-Control`) 외 — 응용 / DB 캐시의 일반 패턴.

---

## 1. 5 가지 전략

| 패턴 | 읽기 | 쓰기 | 일관성 |
| --- | --- | --- | --- |
| **Cache-aside** | 응용이 캐시 확인 → miss 시 DB | 응용이 DB 갱신 + 캐시 무효화 | 약 |
| **Read-through** | 캐시가 DB 조회 자동 | 응용이 DB 직접 | 약 |
| **Write-through** | 캐시 우선 | 캐시 + DB 동시 | 강 |
| **Write-behind** | 캐시 우선 | 캐시 먼저 + DB 비동기 | 약 (delay) |
| **Refresh-ahead** | 캐시 우선 + 만료 전 갱신 | (위 와 결합) | 중 |

---

## 2. Cache-aside (Lazy Loading)

```
[Read]
응용 → 캐시 조회
   ├ Hit → 반환
   └ Miss → DB 조회 → 캐시 저장 → 반환

[Write]
응용 → DB 갱신 → 캐시 invalidate
```

### 코드 예 (Python)

```python
def get_user(user_id):
    cached = redis.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)
    
    user = db.query(User).get(user_id)
    if user:
        redis.setex(f"user:{user_id}", 300, json.dumps(user.to_dict()))
    return user

def update_user(user_id, data):
    db.update(User, user_id, data)
    redis.delete(f"user:{user_id}")    # invalidate
```

### 특징
- 가장 흔함 (Redis + DB 패턴)
- 캐시 장애 시 DB 폴백
- "Lazy" — 첫 요청 시만 캐시
- 미스 시 지연 (Cold cache)

### 함정
- **Cache stampede** — 다수가 동시 miss → DB 폭주
- **TTL** 의 일관성 — DB 변경 후 캐시 stale 가능

---

## 3. Read-through

```
[Read]
응용 → 캐시 layer
   ├ Hit → 반환
   └ Miss → 캐시 가 DB 조회 → 자동 캐시 → 반환

[Write]
응용 → DB 직접 (캐시 layer 거치지 X)
```

### 특징
- 캐시가 능동적 (응용은 캐시만 봄)
- Hibernate Second-level Cache, JCache, EhCache

### 함정
- 쓰기 / 읽기 분리 — 캐시 stale
- 일반적인 데이터 액세스 layer 추가 부담

---

## 4. Write-through

```
[Write]
응용 → 캐시 layer
   ├ 캐시 갱신
   └ 동시에 DB 갱신
```

### 특징
- 캐시와 DB 항상 일관
- 쓰기 latency ↑ (둘 다 대기)
- 강한 일관성 필요 시

### 함정
- DB 쓰기 비싸면 매번 느림
- 캐시에만 있는 데이터 X (모두 DB)

---

## 5. Write-behind (Write-back)

```
[Write]
응용 → 캐시 즉시 갱신
       (DB 는 비동기 / 배치)
```

### 특징
- 쓰기 빠름 (캐시만 즉시)
- DB 부하 ↓ (배치 처리)
- 약한 일관성 (DB 가 늦음)

### 위험
- 캐시 죽으면 미반영 데이터 손실
- 쓰기 순서 / 트랜잭션 어려움

### 사용
- 분석 / 로그 (정확성 < 처리량)
- 좋아요 / 조회수 카운터

---

## 6. Refresh-ahead

```
캐시 entry 만료 전 (예: TTL 80%) 시점에 자동 갱신
사용자는 항상 fresh 캐시
```

### 특징
- 사용자 미스 X
- 백그라운드 갱신 비용
- 예측 가능한 트래픽

### 비교 — Stale-While-Revalidate (HTTP)
- 비슷한 사상 — 만료 후 사용 + 백그라운드 갱신
- HTTP 의 `stale-while-revalidate` directive

---

## 7. 캐시 일관성 패턴

### 7.1 TTL (Time-To-Live)
- 만료 시간 — 자동 무효화
- 가장 단순

### 7.2 Explicit Invalidation
- 변경 시 명시적 삭제
- DB trigger / 응용 코드

### 7.3 Versioning
- 자원의 버전 (ETag / 해시) 으로 키
- 변경 시 새 키 → 옛 자동 expire

### 7.4 Event-driven
- DB 변경 이벤트 → 메시지 큐 → 캐시 invalidate
- Kafka / Redis PubSub
- Debezium (CDC)

---

## 8. Cache Stampede / Thundering Herd

### 문제
```
인기 자원의 TTL 만료
→ 1000 요청이 동시 miss
→ 모두 DB 조회 시도
→ DB 폭주
```

### 해결

#### 8.1 Lock (Mutex)
```python
def get_with_lock(key):
    cached = redis.get(key)
    if cached: return cached
    
    if redis.set(f"lock:{key}", 1, ex=10, nx=True):
        # 잠금 획득 — DB 조회
        value = db.query(...)
        redis.setex(key, 300, value)
        redis.delete(f"lock:{key}")
        return value
    
    # 다른 process 가 갱신 중 — 잠시 대기 후 재시도
    time.sleep(0.1)
    return get_with_lock(key)
```

#### 8.2 Probabilistic Early Expiration (XFetch)
- 만료 전 일정 확률로 미리 갱신
- 단일 요청이 시간 분산

#### 8.3 Refresh-ahead
- 만료 전 능동 갱신
- 미스 자체 회피

---

## 9. Multi-level Cache

```
L1 (응용 메모리) → L2 (Redis) → L3 (DB)
```

### L1 (메모리)
- 가장 빠름 (ns)
- 인스턴스 별 (NUMA 같은 문제)
- 작음 (수십 MB)

### L2 (Redis)
- 빠름 (1 ms)
- 공유
- 큼 (GB)

### L3 (DB)
- 느림 (10-100 ms)
- 영속
- 매우 큼

### 함정
- L1 의 일관성 — 인스턴스 별 다름. Pub/Sub 으로 invalidation 전파.

---

## 10. CDN 캐싱 — HTTP 레벨

자세히 → [[cdn-caching]]

### CDN = 분산 캐시
- 사용자 가까운 Edge
- 정적 자원 / API 응답
- Origin 부하 ↓

### 패턴
- **Tag-based purge** — Surrogate-Key
- **Soft Purge** — stale 유지
- **Cache stamping** — Edge 가 stale 응답 + 백그라운드 갱신

---

## 11. 캐시 키 설계

### 좋은 키
```
user:123                  ← 명확
user:123:profile
user:123:settings:v2
```

### 나쁜 키
```
key1                     ← 의미 없음
GetUserByIdFromDb(123)   ← 너무 김 + 변경 시 모든 키 무효
```

### 패턴
- `<resource>:<id>` — 자원 단위
- `<resource>:<id>:<view>` — 뷰별
- `<resource>:<id>:v<version>` — 버저닝

---

## 12. 함정

### 함정 1 — 캐시 일관성
DB / 캐시 사이 race condition. 항상 가능성 인지.

### 함정 2 — 큰 객체
큰 JSON / 이미지 — 메모리 / 네트워크 부담. 압축 / 분리.

### 함정 3 — Hot key
하나의 인기 키가 캐시 부하 집중. Sharding / Local cache.

### 함정 4 — 캐시 explosion
유저별 모든 query — 캐시 무한. TTL + Max size.

### 함정 5 — 캐시 = 신뢰 X
- 캐시 죽어도 응용 동작해야
- DB 폴백 + Circuit breaker

### 함정 6 — 인증 정보 캐시
사용자 별 응답을 잘못 공유 → 보안 사고. `Vary: Cookie` 또는 `private`.

---

## 13. 학습 자료

- "Caching Strategies" — Microsoft Patterns
- **High Performance Browser Networking** Ch. 11
- "Designing Data-Intensive Applications" (Kleppmann) — 캐시 일관성
- Cloudflare / Fastly 캐싱 가이드

---

## 14. 관련

- [[caching]] — Caching hub
- [[cdn-caching]] — CDN 깊이
- [[cache-control]] — HTTP 의 strategy 표현
- [[../../../distributed-systems/distributed-systems]] — 일관성
