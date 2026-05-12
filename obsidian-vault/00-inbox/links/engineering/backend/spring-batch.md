---
title: "Spring Batch"
kind: knowledge
project: agent-references
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-12T22:30:00+09:00
tags:
  - reference
  - links
  - backend
  - spring-batch
---

# Spring Batch

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 15 링크 |

**[[backend|↑ Backend]]**

> 대용량 배치 처리 / Job / Step / Reader-Writer.

## Reference 링크

- [Spring Batch Reference](https://docs.spring.io/spring-batch/reference/) — 권위 docs
- [Spring Batch Source](https://github.com/spring-projects/spring-batch) — 소스 + 이슈
- [Spring Batch Tutorial (Baeldung)](https://www.baeldung.com/introduction-to-spring-batch) — Batch 입문
- [Spring Batch Listeners](https://docs.spring.io/spring-batch/reference/step/controlling-step-flow.html) — 이벤트 리스너
- [JobRepository / JobLauncher](https://docs.spring.io/spring-batch/reference/job/configuring-launcher.html) — Job 실행 인프라
- [Chunk-oriented Processing](https://docs.spring.io/spring-batch/reference/step/chunk-oriented-processing.html) — chunk vs tasklet
- [Partition / Multi-threaded Step](https://docs.spring.io/spring-batch/reference/scalability.html) — 병렬 처리
- [Restart / Retry / Skip](https://docs.spring.io/spring-batch/reference/step/controlling-step-flow.html#configuring-skip-logic) — 에러 처리
- [Spring Cloud Data Flow](https://docs.spring.io/spring-cloud-dataflow/docs/current/reference/htmlsingle/) — Batch + Stream 오케스트레이션
- [Quartz Scheduler](http://www.quartz-scheduler.org/documentation/) — Spring Batch + 스케줄러
- [Spring Batch + Kubernetes](https://spring.io/blog/2018/12/12/spring-cloud-data-flow-for-pcf-1-7-released) — K8s 운영
- [우아한형제들 - Spring Batch](https://techblog.woowahan.com/?s=spring+batch) — 한국 회사 사례
- [인프런 - 정수원 Spring Batch 강의](https://www.inflearn.com/course/스프링배치) — 한국어 강의
- [Spring Batch Anti-patterns](https://medium.com/@chrisgleissner/anti-patterns-in-spring-batch-c2e5d20d6e07) — Batch 안티패턴
- [Awesome Spring Batch](https://github.com/spring-projects/spring-batch/tree/main/spring-batch-samples) — 공식 sample