---
title: "성능 튜닝 — JVM / DB / Profile"
kind: knowledge
project: agent-references
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-12T20:00:00+09:00
tags:
  - reference
  - links
  - backend
  - performance-tuning
---

# 성능 튜닝 — JVM / DB / Profile

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 15 링크 |

**[[backend|↑ Backend]]**

> JVM heap / GC / DB query / index / profiling 도구 + 실무 패턴.

## Reference 링크

- [JVM Performance — Aleksey Shipilëv](https://shipilev.net/) — JVM 성능 1위 reference
- [Java Mission Control + JFR](https://docs.oracle.com/en/java/javase/21/jfapi/) — Java Flight Recorder
- [async-profiler](https://github.com/async-profiler/async-profiler) — low-overhead Java profiler
- [VisualVM](https://visualvm.github.io/) — JVM 모니터링 도구
- [GC Tuning Guide (Oracle)](https://docs.oracle.com/en/java/javase/21/gctuning/) — 공식 GC 튜닝
- [Use The Index, Luke! (DB index)](https://use-the-index-luke.com/) — DB 인덱스 성능
- [Postgres pg_stat_statements](https://www.postgresql.org/docs/current/pgstatstatements.html) — 쿼리 성능 분석
- [MySQL Performance Schema](https://dev.mysql.com/doc/refman/8.0/en/performance-schema.html) — MySQL 성능 추적
- [JMH (Java Microbenchmark Harness)](https://github.com/openjdk/jmh) — 벤치마크 표준
- [Flame Graphs (Brendan Gregg)](https://www.brendangregg.com/flamegraphs.html) — 성능 시각화
- [Brendan Gregg blog](https://www.brendangregg.com/) — Linux / 성능 권위
- [Designing Data-Intensive Applications — Storage 성능](https://dataintensive.net/) — DB 성능 + 시스템 성능
- [Effective Java — Performance items](https://github.com/HugoMatilla/Effective-JAVA-Summary) — 67-71 성능 항목
- [Java Performance (책 — Scott Oaks)](https://www.oreilly.com/library/view/java-performance-2nd/9781492056102/) — JVM 성능 표준 책
- [우아한형제들 - 성능 튜닝 글](https://techblog.woowahan.com/?s=performance) — 한국 회사 사례
