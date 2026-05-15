---
title: "GC analysis — G1GC / ZGC / log 분석"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:19:00+09:00
tags: [devops, performance, gc]
---

# GC analysis — G1GC / ZGC / log 분석

**[[performance|↑ performance]]**

---

## 1. GC log 켜기 (★ 필수)

```bash
# JDK 9+ unified logging
-Xlog:gc*=info:file=/var/log/gc.log:time,uptime,level,tags:filecount=10,filesize=100M

# JDK 8 legacy
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-Xloggc:/var/log/gc.log
-XX:+UseGCLogFileRotation
-XX:NumberOfGCLogFiles=10
-XX:GCLogFileSize=100M
```

→ 모든 production = GC log 보존. 분석 시 필수.

---

## 2. log 분석 도구

| | 무엇 |
| --- | --- |
| **GCViewer** | OSS, 단순 |
| **GCEasy** (★) | 무료 SaaS, AI 분석 |
| **JClarity Censum** | 상용 |
| **Mission Control + JFR** | JDK native |
| **gceasy.io** | upload + 자동 분석 |

→ 무료 = **GCEasy** 권장.

---

## 3. G1GC log 읽기

```
[3.456s][info][gc] GC(15) Pause Young (Normal) (G1 Evacuation Pause) 1024M->512M(2048M) 50.123ms
  │       │      │    │            │              │              │                       │
  │       │      │    │            │              │              │                       └─ 시간 (★ 가장 중요)
  │       │      │    │            │              │              └─ heap 의 전체 (max)
  │       │      │    │            │              └─ heap used 직후
  │       │      │    │            └─ heap used 직전 → 512MB 회수
  │       │      │    └─ G1 의 evacuation 단계
  │       │      └─ Normal Young GC (vs Mixed / Full)
  │       └─ GC ID (★ 추적용)
  └─ uptime (3.456s 이후)
```

---

## 4. 좋은 vs 나쁜 GC

```
좋은 (G1GC):
  Pause Young 30-100ms
  Concurrent Cycle 의 STW phase 50ms 이내
  Young GC 빈도 적당 (10s-60s)
  Mixed GC 가끔 (Old gen 정리)
  Full GC 없음 (★)

나쁜:
  Full GC 빈번 (1초+ pause)
  Young GC > 200ms
  Concurrent Cycle 의 fail (To-Space exhaustion)
  Allocation Stall (ZGC)
  GC overhead > 10%
```

---

## 5. throughput vs latency tradeoff

```
GC time / total time:
  > 10%   = 심각 (app 시간 < 90%)
  5-10%   = 검토
  < 5%    = 정상 (대부분)
  
계산:
  total GC time = log 의 모든 pause 합
  total = uptime
  
GCEasy 의 "GC Throughput" 지표.
```

---

## 6. 흔한 문제 패턴

### A. heap 부족 → Full GC 폭주

```
[GC ... ] Pause Full (G1 Compaction Pause) 2048M->1900M(2048M) 5000ms
```

→ heap 의 95% 사용. Full GC 가 짧은 시간 회수.
→ 해결: heap ↑ 또는 메모리 leak 분석.

### B. allocation rate 너무 빠름

```
짧은 시간 GC 빈번:
  3.0s: Young GC
  3.5s: Young GC
  4.0s: Young GC
  ...
```

→ Young Gen 가 빠르게 fill.
→ 해결: Young Gen ↑ (-XX:G1NewSizePercent=30) 또는 allocation 줄임.

### C. promotion failure

```
[GC] To-Space Exhausted   ← 위험!
```

→ Survivor → Old 못 가서 fail.
→ 해결: G1ReservePercent ↑.

### D. humongous allocation

```
[GC] Humongous allocation requested
```

→ region size 의 50% 초과 객체. byte[] / String 등 큰 객체.
→ 해결: G1HeapRegionSize ↑ 또는 큰 객체 회피.

### E. Allocation Stall (ZGC)

```
[gc] Allocation Stall (worker-1) 25.123ms
```

→ thread 가 GC 기다림. RAM 부족 / GC 따라잡기 못함.
→ 해결: heap ↑ 또는 ConcGCThreads ↑.

---

## 7. heap dump 분석 (Eclipse MAT)

```
1. heap dump 만들기
   jcmd <pid> GC.heap_dump /tmp/heap.hprof
   # OOM 시 자동: -XX:+HeapDumpOnOutOfMemoryError

2. Eclipse MAT 열기
   - Leak Suspects Report
   - Dominator Tree
   - Histogram (instance 별)
   - Path to GC Roots (왜 GC 안 됨)

3. 흔한 leak 원인:
   - static collection / map
   - listener / callback unregister 안 함
   - ThreadLocal cleanup 안 함
   - Hibernate session 의 cache
   - JDBC connection 누수
```

---

## 8. live monitoring (★)

```bash
# jstat (실시간)
jstat -gc <pid> 1000

#  S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT
#  0.0   1024.0  0.0   1024.0  10240.0   2048.0   30720.0    15360.0  6400.0 6100.0  768.0  730.0   45    1.234     2    0.500    1.734

# 의미:
# S0C/S1C: Survivor 0/1 capacity
# EC/EU:   Eden capacity / used
# OC/OU:   Old capacity / used
# YGC:     Young GC count
# FGC:     Full GC count (★ 0 이어야)
# YGCT:    Young GC total time
# FGCT:    Full GC total time
# GCT:     모든 GC total time
```

```bash
# Prometheus / Micrometer
jvm_gc_pause_seconds_count
jvm_gc_pause_seconds_sum
jvm_memory_used_bytes{area="heap", id="G1 Old Gen"}
```

---

## 9. tuning iteration

```
1. 현재 GC 분석 (log + dump)
2. hypothesis (heap ↑ / GC 알고리즘 / 옵션)
3. 한 변경만
4. load 후 측정
5. better? worse? → iterate

❌ 여러 옵션 동시 변경 — 어느 게 효과인지 모름
```

---

## 10. ZGC tuning

```bash
# ZGC (JDK 17+)
-XX:+UseZGC
-XX:+UnlockExperimentalVMOptions
-XX:ZAllocationSpikeTolerance=2.0    # spike 허용
-XX:ConcGCThreads=8                  # concurrent GC thread
-XX:ParallelGCThreads=16

# generational ZGC (JDK 21+)
-XX:+UseZGC
-XX:+ZGenerational
```

→ ZGC 는 pause < 10ms 보장. 단점: throughput 약간 ↓.

---

## 11. Shenandoah

```bash
# RedHat OpenJDK
-XX:+UseShenandoahGC
-XX:ShenandoahGCMode=adaptive
```

→ ZGC 와 비슷. RedHat 환경.

---

## 12. Loom (virtual thread, JDK 21+)

```java
// 전통 thread pool
ExecutorService es = Executors.newFixedThreadPool(200);

// virtual thread (★ blocking I/O 적합)
ExecutorService es = Executors.newVirtualThreadPerTaskExecutor();
```

→ thread 가 cheap (수만 thread OK). connection pool 의 의미 변경.

→ GC 가 thread stack 의 관리도 효율 ↑.

---

## 13. 함정

1. **GC log 없이 분석** — 추측만.
2. **Xmx 늘리면 해결 가정** — 진짜 leak 인지.
3. **Full GC 무시** — 즉시 분석.
4. **Survivor 너무 큼** — 짧은 lifecycle 객체 가 Old 로 promotion.
5. **여러 옵션 동시 변경** — A/B 못 함.
6. **dev / staging GC ≠ prod** — load 다름. prod log 분석.
7. **GraalVM native 의 GC** — Serial-like, 작은 service 만.

---

## 14. 관련

- [[performance|↑ performance]]
- [[jvm-tuning]]
- [[profiling]]
- [[../monitoring/application-metrics|↗ JVM metric]]
