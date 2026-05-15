---
title: "Caching strategies — Redis / Caffeine / invalidation"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:31:00+09:00
tags: [devops, performance, cache, redis, caffeine]
---

# Caching strategies — Redis / Caffeine / invalidation

**[[performance|↑ performance]]**

---

## 1. cache 가 답인 경우

```
1. read-heavy (DB read 비싸)
2. computed result (heavy 계산)
3. external API (rate limit / latency)
4. static / 잘 변하지 않는 data
5. session / temporary state
```

→ "write 매우 빈번" / "consistency 절대" = cache 안 좋음.

---

## 2. layer

```
[Client] 
   ↓ CDN cache              (Cloudflare / CloudFront)
[edge]
   ↓ application gateway cache (nginx proxy_cache)
[app server]
   ↓ in-memory (★ Caffeine)
   ↓ distributed (★ Redis / Memcached)
   ↓ DB query cache (materialized view)
[DB]
   ↓ buffer pool (PG shared_buffers)
```

→ 가장 가까운 layer 에서 hit = 빠름.

---

## 3. patterns (★)

### A. cache-aside (lazy load) — 가장 흔함

```java
public Order get(Long id) {
    Order order = cache.get(id);
    if (order == null) {
        order = db.find(id);
        if (order != null) {
            cache.set(id, order, TTL);
        }
    }
    return order;
}
```

→ 단순. cache fail 시 DB 가 backup.

### B. read-through

```
app → cache layer → DB (cache 가 자동 fetch)

도구: Hibernate 2nd-level cache, Spring Cache
```

→ application code 단순. 단 cache layer 가 DB 연결 필요.

### C. write-through

```
write 시 cache + DB 동시:
  app → cache (write) → DB (write)
  
→ consistency 좋음. write 느림.
```

### D. write-behind (write-back)

```
write 시 cache 만 → queue → DB 비동기:
  app → cache (write)
              ↓ queue
            DB (write 비동기)
            
→ write 빠름. data loss 위험.
```

### E. refresh-ahead

```
TTL 이전에 미리 refresh:
  Caffeine 의 refreshAfterWrite
  
→ cache miss 거의 X. 단 background load.
```

---

## 4. Caffeine (★ in-memory)

```java
LoadingCache<String, Order> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(Duration.ofMinutes(10))
    .refreshAfterWrite(Duration.ofMinutes(5))
    .recordStats()
    .build(key -> db.find(key));   // miss 시 자동

Order order = cache.get("123");

// stat
cache.stats();   // hit rate / miss / load time
```

→ JVM heap 안. 매우 빠름 (nanoseconds). 단 단일 instance.

```
크기 결정:
  - heap 의 10-30%
  - key + value 의 평균 크기 계산
  - W-TinyLFU 알고리즘 (Caffeine default — LRU 보다 좋음)
```

---

## 5. Redis (distributed)

```java
// Spring Data Redis
@Cacheable(value = "orders", key = "#id")
public Order get(Long id) {
    return repo.findById(id).orElseThrow();
}

@CachePut(value = "orders", key = "#order.id")
public Order update(Order order) {
    return repo.save(order);
}

@CacheEvict(value = "orders", key = "#id")
public void delete(Long id) {
    repo.deleteById(id);
}
```

```redis
# 직접
SET order:123 '{"id":123, ...}' EX 600    # 10분 TTL
GET order:123
DEL order:123

# pipeline
MGET order:123 order:124 order:125

# hash (큰 object 의 일부만 fetch)
HSET user:123 name "alice" email "a@b.com"
HGET user:123 email
```

---

## 6. TTL 전략

```
짧음 (1min):    자주 변경 / 실시간
중간 (10min):   일반 비즈니스 data
긺 (1h+):       static (config / 카탈로그)
없음 (영구):    immutable (image / lookup table)

→ data 별 합리적 TTL. 절대 외운 값 X.
```

```java
// adaptive TTL
int ttl = isHot ? 30 : 600;   // hot key 짧게 (freshness)
```

---

## 7. cache invalidation (★ 핵심 어려움)

> "There are only two hard things in CS: cache invalidation and naming things." — Phil Karlton

```
방법:

A. TTL only — 결국 만료. consistency 약함.
B. delete on write — 직접 delete (가장 흔함)
   - DB 변경 시 cache delete
   - 위험: race condition (next read 가 stale 직전 write)
   
C. version key — key 에 version 포함
   user:123:v5 → user:123:v6
   
D. event-driven (CDC)
   DB change → Kafka → cache update
   
E. write-through — 항상 fresh
```

```java
// 표준 패턴 (cache-aside + invalidate on write)
@Transactional
public Order update(Long id, OrderUpdate req) {
    Order order = repo.findById(id).orElseThrow();
    order.apply(req);
    Order saved = repo.save(order);
    
    // cache 삭제 (다음 read 가 fresh)
    cache.evict("order:" + id);
    return saved;
}
```

---

## 8. cache stampede (thundering herd ★)

```
hot key 의 TTL 만료 직후:
  100 request 가 동시에 cache miss
  → 100 동시 DB query
  → DB 부담

해결:
  A. mutex / single-flight
  B. soft TTL + hard TTL (refresh-ahead)
  C. probabilistic early expiration
  D. background refresh job
```

```java
// single-flight (Caffeine 의 default)
LoadingCache<String, Order> cache = Caffeine.newBuilder()
    .build(key -> db.find(key));
// → 동시 미스 → 한 thread 만 load, 나머지 wait
```

```redis
# Redis lock 으로 stampede 방지
SET lock:order:123 1 NX EX 10
# only 한 thread 만 success → DB query → cache set → release
```

---

## 9. hot key

```
1 key 에 100k+ req/s → 한 Redis shard 부담

해결:
  - shard key (key + random suffix)
  - local cache (Caffeine) 앞에
  - read replica
  - rate limit
```

```java
// shard key
String key = "popular:" + (id % 16);   // 16 shard
```

---

## 10. cache 의 size estimate

```
1M user × 1KB / user = 1GB
+ overhead (Redis 30%) = 1.3GB

→ Redis instance 의 maxmemory 설정.
→ eviction policy: allkeys-lru / volatile-lru / allkeys-lfu (★)
```

---

## 11. observability

```
metric:
  - hit rate (★ 가장 중요)
  - miss rate
  - eviction count
  - average load time (miss 시 DB query)
  - memory usage
  - 가장 hot key

hit rate < 80%? → review:
  - TTL 너무 짧음?
  - 잘못된 cache key?
  - 사용자 다양함 (long-tail)?
  - working set 이 메모리 보다 큼?
```

```promql
# Prometheus
sum(rate(cache_hits_total[5m])) / 
sum(rate(cache_requests_total[5m]))
```

---

## 12. 분산 cache 결정

```
Redis vs Memcached:

Redis:
  + 다양 data type (string/hash/list/set/sorted-set)
  + persistence (AOF / RDB)
  + replication / cluster
  + pub/sub / streams
  + Lua scripting

Memcached:
  + 매우 빠름 (단순)
  + multi-thread (Redis 6+ 도 IO multi-thread)
  - data type 단순
  - persistence X

→ 거의 항상 Redis. Memcached = legacy / 특수 케이스.
```

---

## 13. CDN cache

```
HTTP 의 Cache-Control:
  public, max-age=86400              # 1 day public
  private, max-age=300                # 5min, user 전용
  no-cache, must-revalidate           # 항상 검증
  no-store                            # 절대 X (PII)
  immutable                           # 영구 (hashed assets)
```

→ static asset 의 hash 파일명 + `immutable` = 최고.

---

## 14. anti-patterns (★)

```
1. 모든 것 cache → memory 부족
2. 큰 객체 cache → 1 fetch 도 느림 (~1MB+)
3. 짧은 TTL + 큰 miss → DB 부담
4. invalidation 안 함 → stale data
5. user-specific cache → low hit
6. cache 가 DB 보다 큼 → 의미 X
7. 모든 read 가 cache → cold start 매번
```

---

## 15. 함정

1. **TTL 만 의존** — 변경 즉시 반영 X.
2. **cache stampede** — TTL 만료 폭주.
3. **hot key 무시** — 한 shard 부담.
4. **eviction policy 잘못** — 자주 쓰는 것 evict.
5. **persistence 없음 + restart** — 모두 cold.
6. **monitoring 없음** — hit rate 모름.
7. **PII cache** — GDPR / 보안.
8. **stale 의 영향 평가 안 함** — 결제 / 재고 ≠ catalog.

---

## 16. 관련

- [[performance|↑ performance]]
- [[database-performance]]
- [[../../60-recipes/spring/cache-redis/cache-redis|↗ Redis cache recipe]]
