---
title: "Profiling — async-profiler / JFR / flame graph"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:21:00+09:00
tags: [devops, performance, profiling, flame-graph]
---

# Profiling — async-profiler / JFR / flame graph

**[[performance|↑ performance]]**

---

## 1. profiling 의 종류

```
sampling: stack 정기 snapshot (low overhead)
instrumentation: 모든 호출 trace (high overhead)

대상:
  CPU         — 어디서 시간 보냄
  Memory     — 누가 할당
  Lock       — 어디서 대기
  IO         — disk / network 시간
  GC         — GC 영향
```

→ production = sampling. dev = instrumentation OK.

---

## 2. async-profiler (★ 표준)

```bash
# 설치 (Linux)
wget https://github.com/async-profiler/async-profiler/releases/download/v3.0/async-profiler-3.0-linux-x64.tar.gz
tar xzf async-profiler-3.0-linux-x64.tar.gz

# CPU profile (60초)
./bin/asprof -d 60 -f /tmp/profile.html <pid>

# allocation profile
./bin/asprof -d 60 -e alloc -f /tmp/alloc.html <pid>

# lock profile
./bin/asprof -d 60 -e lock -f /tmp/lock.html <pid>

# 결과 = flame graph (HTML 직접)
```

→ **0.5% 이하 overhead**. production OK.

→ kernel-level perf event 사용 → 더 정확.

---

## 3. JFR (Java Flight Recorder, ★ JDK native)

```bash
# 항상 켜기 (low overhead 1-2%)
-XX:StartFlightRecording=duration=60s,filename=/tmp/profile.jfr,settings=profile

# 또는 runtime
jcmd <pid> JFR.start duration=60s filename=/tmp/profile.jfr

# 지속 (rolling buffer)
jcmd <pid> JFR.start name=continuous maxsize=200MB maxage=1h
jcmd <pid> JFR.dump name=continuous filename=/tmp/profile.jfr

# 분석:
#   - JDK Mission Control (JMC) 무료
#   - IntelliJ 의 profiler
#   - upload to JFR viewer
```

→ JDK 8u272+ free. production 권장.

---

## 4. flame graph 해석

```
Y축:  stack depth (위 = 깊은 call)
X축:  CPU time 비율 (넓을수록 시간 많이)

읽는 법:
  1. 가장 넓은 박스 = hot spot
  2. 박스가 평평 = 그 함수 자체에 시간
  3. 박스 위 cascade = sub-call
  4. 색깔 random (구분만)

목표:
  높은 박스 = 정상 (call depth)
  평평한 박스 = 의심 (자체 시간 많이)
```

→ 처음엔 어색하지만 익숙해지면 즉시 패턴 보임.

---

## 5. CPU profile 흔한 패턴

```
1. JSON serialization (Jackson)
   → 큰 객체, frequent → cache 또는 다른 lib

2. log4j layout formatting
   → toString() 의 expensive op

3. regex 컴파일
   → static / lazy init

4. Stream API 의 box/unbox
   → primitive stream 사용

5. HashMap resize
   → 초기 capacity

6. SSL handshake
   → connection pool / keepalive

7. DB query 의 PreparedStatement re-create
   → pool 의 PreparedStatement cache
```

---

## 6. memory allocation profile

```bash
./bin/asprof -d 60 -e alloc -f /tmp/alloc.html <pid>
```

찾는 것:
- 큰 byte[] / String 의 frequent
- autoboxing (Integer, Long)
- DateFormat re-creation
- StringBuilder 안 쓰는 concat
- 큰 collection 의 resize

```
큰 객체 자주 생성:
  → cache / pool
  → reuse buffer
  → static format
```

---

## 7. lock contention

```bash
./bin/asprof -d 60 -e lock -f /tmp/lock.html <pid>
```

찾는 것:
- synchronized block 의 wait
- ReentrantLock 의 contention
- ConcurrentHashMap 의 bucket lock

해결:
- shard data
- lock-free (Atomic / CAS)
- ConcurrentLinkedQueue
- ReadWriteLock 사용

---

## 8. continuous profiling (★ 시니어)

```
production 에서 지속:
  - Datadog Continuous Profiler
  - Pyroscope (OSS)
  - Grafana Phlare / Pyroscope
  - Sentry Profiling
  - New Relic
  - Polar Signals (eBPF)

장점:
  - "어제 14시 spike 때 뭐가 느렸나" 사후 분석
  - flame graph 시간 흐름
  - rollback 의 비교 (before / after)
```

```bash
# Pyroscope 설치
helm install pyroscope grafana/pyroscope -n pyroscope --create-namespace

# Java agent
-javaagent:./pyroscope.jar \
-Dpyroscope.application.name=my-app \
-Dpyroscope.server.address=http://pyroscope:4040
```

---

## 9. perf (Linux 의 표준)

```bash
# system-wide
sudo perf record -F 99 -p <pid> -g -- sleep 60

# call graph 분석
sudo perf report

# flame graph
sudo perf script | \
    ./FlameGraph/stackcollapse-perf.pl | \
    ./FlameGraph/flamegraph.pl > flame.svg
```

→ Java + native 통합 (JIT symbol 도 보임 with `-XX:+PreserveFramePointer`).

---

## 10. eBPF (★ 현대)

```bash
# BCC tools
sudo profile-bpfcc -p <pid> 30
sudo offcpu-bpfcc 30      # 어디서 wait
sudo memleak-bpfcc -p <pid>

# Cilium Tetragon — security observability
# Polar Signals — continuous + eBPF
# bpftrace
```

→ kernel-level profiling. very low overhead.

---

## 11. distributed tracing 과의 관계

```
profiling:    한 process 의 hot spot
tracing:      여러 service 의 request 흐름

같이 사용:
  trace 가 "이 request 가 200ms" 알려줌
  profile 가 "그 200ms 의 80% 가 JSON parse" 알려줌
```

---

## 12. database profile

```sql
-- Postgres
EXPLAIN ANALYZE SELECT ...        -- 실제 실행 + 시간

-- slow query log
ALTER SYSTEM SET log_min_duration_statement = 1000;   -- 1s+
SELECT pg_reload_conf();

-- 활성 query
SELECT pid, state, wait_event, query
FROM pg_stat_activity
WHERE state != 'idle';

-- 통계
SELECT * FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;
```

---

## 13. HTTP profiling

```bash
# curl timing
curl -w "@curl-format.txt" -o /dev/null -s https://api.example.com

# 또는 lazy
curl -w '\n  namelookup: %{time_namelookup}s\n  connect: %{time_connect}s\n  ttfb: %{time_starttransfer}s\n  total: %{time_total}s\n' \
     https://api.example.com
```

```
time_namelookup:  DNS
time_connect:     TCP handshake
time_appconnect:  TLS handshake
time_pretransfer: ready to send
time_starttransfer: TTFB (Time To First Byte)
time_total:       전체
```

---

## 14. 함정

1. **profile 안 하고 추측** — 80% 시간 다른 곳.
2. **dev / staging profile** — load 다름, 결과 의미 X.
3. **instrumentation in prod** — 5-30% overhead.
4. **flame graph 해석 못함** — 학습 곡선.
5. **profile 만 보고 fix** — measure → fix → measure 다시.
6. **GC log 와 같이 안 봄** — GC pause 가 profile 에 안 나옴.
7. **continuous profiling 비용** — 큰 traffic 시 sampling rate 검토.

---

## 15. 관련

- [[performance|↑ performance]]
- [[jvm-tuning]]
- [[gc-analysis]]
- [[../monitoring/opentelemetry|↗ OTel]]
