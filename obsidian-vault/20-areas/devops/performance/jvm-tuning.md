---
title: "JVM tuning — heap / GC / JIT / container"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:17:00+09:00
tags: [devops, performance, jvm, java]
---

# JVM tuning — heap / GC / JIT / container

**[[performance|↑ performance]]**

---

## 1. JVM 메모리 구조

```
JVM Process Memory:
  ├─ Heap                 (Xmx 으로 제어)
  │   ├─ Young Gen
  │   │   ├─ Eden
  │   │   └─ Survivor (S0/S1)
  │   └─ Old Gen (Tenured)
  ├─ Metaspace            (class metadata)
  ├─ Code Cache           (JIT)
  ├─ Compressed Class Space
  ├─ Thread Stacks        (스레드 별 ~1MB)
  ├─ Direct Buffer        (NIO)
  ├─ Native (JNI)
  └─ GC overhead
```

→ heap 만 줄여도 안 됨. native + thread stack + 등 별도 cost.

---

## 2. heap 크기 결정 (★)

```
원칙:
  Xmx = container memory × 50-75%
  
예 (container 1Gi):
  - JVM 21+: -XX:MaxRAMPercentage=75
  - JVM 8/11 manual: -Xmx768m

container memory 1Gi 의 분배:
  Heap (Xmx)              : 512Mi (50%)
  Metaspace               : 100Mi
  Code cache              : 50Mi
  Thread stacks (100×1MB)  : 100Mi
  Direct buffer / 기타       : 200Mi
  여유 (kernel / OOM 방지)   : 100Mi
```

→ Xmx = container memory 동일 = OOM kill 발생.

---

## 3. container-aware (★ JDK 10+)

```bash
# JDK 8u131+ 부터 cgroup 인식
-XX:+UseContainerSupport

# JDK 10+ default

# JDK 8 < 131 = manual
-Xmx<value>          # 직접
```

```bash
# 추천 (JDK 11+)
-XX:InitialRAMPercentage=50
-XX:MaxRAMPercentage=75
-XX:MinRAMPercentage=50
```

→ container limit 변경 시 자동 조정.

---

## 4. GC algorithm 선택

| GC | 무엇 | 사용 |
| --- | --- | --- |
| **Serial** | 단일 thread | 작은 heap (< 100MB) |
| **Parallel** | multi thread, throughput | batch / 단순 |
| **CMS** | concurrent (deprecated, JDK 14 제거) | (사용 X) |
| **G1GC** | balanced, pause ~ 100ms | default JDK 9+ |
| **ZGC** | ultra-low pause (< 10ms) | 큰 heap (10GB+) |
| **Shenandoah** | ultra-low pause | RedHat |
| **Epsilon** | no-op GC | 테스트 |

→ **JDK 11-17 = G1GC, JDK 21 = ZGC** (production ready).

---

## 5. G1GC 권장 옵션

```bash
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200          # 목표 (best effort)
-XX:G1HeapRegionSize=16M           # heap region (4MB-32MB)
-XX:InitiatingHeapOccupancyPercent=45   # 45% 차면 concurrent start
-XX:G1ReservePercent=15            # 예약 (full GC 방지)

# 옛 GC log → 새 unified log
-Xlog:gc*=info:file=/var/log/gc.log:time,uptime,level,tags:filecount=10,filesize=100M
```

---

## 6. ZGC (JDK 17+ production)

```bash
-XX:+UseZGC
-Xlog:gc*=info:file=/var/log/gc.log:time
```

특징:
- pause < 10ms 항상
- heap size 거의 무관 (8MB ~ 16TB)
- multi-region 처리
- JDK 17 부터 production-ready
- JDK 21 = generational ZGC (★ 더 빠름)

```bash
# generational ZGC (JDK 21+, 권장)
-XX:+UseZGC
-XX:+ZGenerational
```

---

## 7. metaspace

```bash
-XX:MetaspaceSize=128m              # 초기
-XX:MaxMetaspaceSize=256m           # 최대 (OOM 방지)

# 모니터
jcmd <pid> VM.metaspace
```

→ class loader 누수 시 폭증 (Spring Boot reload, dynamic class).

---

## 8. thread 설정

```bash
-Xss512k                              # thread stack (default 1MB)
# 많은 thread (1000+) 시 줄임

# JIT compiler thread
-XX:CICompilerCount=4

# parallel GC thread
-XX:ParallelGCThreads=4
-XX:ConcGCThreads=2
```

→ container 의 CPU = 2 면 JVM 가 잘못 인식 (host 16 core 로) → manual.

---

## 9. Spring Boot 표준 옵션

```bash
java \
    -server \
    -Xms512m -Xmx1g \                    # 또는 percentage
    -XX:+UseG1GC \
    -XX:MaxGCPauseMillis=200 \
    -XX:MetaspaceSize=128m \
    -XX:MaxMetaspaceSize=256m \
    -XX:+UseStringDeduplication \
    -XX:+UnlockExperimentalVMOptions \
    -XX:+UseContainerSupport \
    -XX:MaxRAMPercentage=75 \
    -Xlog:gc*=info:file=/var/log/gc.log:time:filecount=10,filesize=100M \
    -XX:+HeapDumpOnOutOfMemoryError \
    -XX:HeapDumpPath=/var/log/heapdump.hprof \
    -XX:+ExitOnOutOfMemoryError \         # OOM 시 즉시 종료 (k8s 재시작)
    -Djava.security.egd=file:/dev/./urandom \
    -jar app.jar
```

---

## 10. JIT tuning (★ 자주 잊음)

```
JIT:
  C1 compiler — 빠른 tier (간단)
  C2 compiler — 최적 tier (오래 걸림)

cold start:
  처음 N 요청 = 느림 (interpreter 만)
  → warm-up 후 점차 빨라짐

옵션:
  -XX:+TieredCompilation               # default
  -XX:TieredStopAtLevel=1              # quick start, 단 max perf 안 됨
```

→ Lambda / cold start = warm-up 중요. CDS (Class Data Sharing) 활용.

---

## 11. AppCDS (★ 빠른 startup)

```bash
# 1. CDS archive 생성
java -XX:ArchiveClassesAtExit=app.jsa -jar app.jar

# 2. 사용
java -XX:SharedArchiveFile=app.jsa -jar app.jar

# 30-40% startup 단축. cold start 시 큼.
```

---

## 12. GraalVM Native Image (★)

```
JAR (JVM):  startup 3-10s, RSS 200-400MB
Native:     startup 50ms, RSS 50-100MB

→ Lambda / CLI / cold start critical 면 큰 이득.

단점:
  - build 시간 ↑ (5-15분)
  - reflection / dynamic class loading 제약
  - debug 어려움
  - Spring Boot 3+ 가 native 지원
```

---

## 13. 진단 명령

```bash
# heap usage
jcmd <pid> GC.heap_info
jmap -heap <pid>
jstat -gc <pid> 1000          # 1초마다 GC stat

# 또는 modern
jcmd <pid> GC.heap_dump /tmp/heap.hprof
jcmd <pid> Thread.print

# JFR (Flight Recorder, JDK 8u272+ free)
jcmd <pid> JFR.start duration=60s filename=/tmp/profile.jfr
# Mission Control 으로 분석

# native memory tracking
jcmd <pid> VM.native_memory summary
```

---

## 14. 메모리 leak 분석

```
heap dump (.hprof) 분석:
  - Eclipse MAT (★)
  - VisualVM
  - JProfiler / YourKit (상용)

찾는 것:
  - Dominator Tree
  - Largest Objects
  - Leak Suspects
  - Histogram (instance count)
```

---

## 15. Spring 특화 tuning

```yaml
# application.yml
server:
  tomcat:
    threads:
      max: 200
      min-spare: 20
    max-connections: 8192
    accept-count: 100

spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 3000
      idle-timeout: 600000
      max-lifetime: 1800000
```

→ thread / connection pool 의 balance 가 사실 더 큰 영향.

---

## 16. 함정

1. **Xmx = container memory** — OOM kill.
2. **container-aware off (JDK 8 < 131)** — host RAM 으로 인식.
3. **ParallelGC 큰 heap** — long pause.
4. **GC log 안 켬** — 분석 불가.
5. **heap dump 안 만듦 (OOM 시)** — `+HeapDumpOnOutOfMemoryError`.
6. **thread pool 너무 큼** — context switch 폭주.
7. **JIT warm-up 무시** — readiness 너무 빨리.
8. **CDS / Native 검토 안 함** — startup 너무 느림.

---

## 17. 관련

- [[performance|↑ performance]]
- [[gc-analysis]]
- [[profiling]]
- [[../docker/dockerfile-best-practices|↗ container]]
