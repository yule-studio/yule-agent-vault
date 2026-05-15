---
title: "Performance engineering — JVM / system / profiling ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:15:00+09:00
tags: [area, devops, performance]
---

# Performance engineering — JVM / system / profiling ★

**[[../devops|↑ devops]]**

---

## 1. 왜

```
DevOps 시니어 = 어디서 느린지 분석 가능해야:
  - 새로 deploy 후 latency 증가
  - traffic spike 시 OOM
  - DB query 가 느림
  - GC pause spike
  - thread pool 고갈

→ application + system + infra 전 layer 디버깅.
```

→ "system 이 느린데 왜?" 가 시니어의 질문.

---

## 2. 4 layer 진단

```
1. Application
   - GC / heap
   - thread / lock
   - 알고리즘 / DB query

2. JVM / runtime
   - JIT
   - GC algorithm
   - thread pool

3. OS / kernel
   - CPU / memory / IO / network
   - cgroup limit
   - syscall

4. Hardware / Cloud
   - instance type
   - NUMA
   - SSD / network speed
```

→ "어느 layer 가 bottleneck?" 부터.

---

## 3. 하위 영역

- [[jvm-tuning]] — heap / GC / JIT / Spring Boot
- [[gc-analysis]] — G1GC / ZGC / Shenandoah / GC log 분석
- [[profiling]] — async-profiler / flame graph / JFR
- [[system-performance]] — CPU / IO / network 분석 (Brendan Gregg)
- [[load-testing]] — k6 / JMeter / wrk / Locust
- [[database-performance]] — Postgres / MySQL 튜닝 / index / explain
- [[capacity-planning]] — load forecast / right-sizing
- [[caching-strategies]] — Redis / Caffeine / cache invalidation
- [[bottleneck-patterns]] — N+1 / hot key / thread saturation 등
- [[chaos-performance]] — load + chaos 동시
- [[pitfalls]]

---

## 4. USE method (★ Brendan Gregg)

```
모든 resource 마다:
  U - Utilization        % busy
  S - Saturation         queue / wait
  E - Errors             count

resource:
  - CPU (per-core)
  - Memory (capacity, swap)
  - Disk (IO, capacity)
  - Network (bandwidth, errors)
  - Process / kernel
```

→ 60초 안에 어디 bottleneck.

---

## 5. RED method (request-based)

```
모든 service:
  R - Rate              requests / sec
  E - Errors            error count / rate
  D - Duration          p50 / p95 / p99 latency

→ 사용자 경험 직결.
```

---

## 6. 시니어의 단골 질문

```
"느리다"
  → 1. 평균 vs p99 어느 쪽? (p99 가 진짜)
  → 2. 항상 vs 특정 시간? (peak / 특정 endpoint?)
  → 3. cold start vs warm?
  → 4. 외부 dep 영향? (DB / Redis / 외부 API)
  → 5. resource saturation? (CPU / Memory / IO / Network)
  → 6. GC pause? thread block? lock contention?

"메모리 사용 ↑"
  → heap vs off-heap?
  → leak vs 큰 cache?
  → JNI / direct buffer?

"deploy 후 traffic 못 받음"
  → readiness probe 늦음?
  → warm-up?
  → connection pool warmup?
```

---

## 7. 학습 순서

1. Day 1: [[system-performance]] (top / vmstat / iostat 60초)
2. Day 2: [[jvm-tuning]] + [[gc-analysis]]
3. Day 3: [[profiling]] (flame graph)
4. Day 4: [[database-performance]] (EXPLAIN / index)
5. Day 5: [[load-testing]] + [[capacity-planning]]

---

## 8. 추천 도서 (★)

- **Systems Performance** (Brendan Gregg) — bible
- **BPF Performance Tools** (Brendan Gregg) — eBPF
- **Java Performance Companion** (Charlie Hunt) — JVM
- **Database Internals** (Alex Petrov) — DB
- **Designing Data-Intensive Applications** (Kleppmann) — system

---

## 9. 관련

- [[../devops|↑ devops]]
- [[../linux/performance-troubleshooting|↗ linux performance]]
- [[../monitoring/application-metrics|↗ application metrics]]
- [[../monitoring/slos-and-sli|↗ SLO]]
