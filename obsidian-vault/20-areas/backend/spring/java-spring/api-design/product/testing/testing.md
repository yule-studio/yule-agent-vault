---
title: "product testing hub"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:50:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - testing
---

# product testing hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[../product|↑ hub]]**

> TDD 기반 — image Project 9 의 핵심 기술 매핑.

---

## 1. 영역

| 노트 | 영역 |
| --- | --- |
| [[test-scenarios]] | AC → test case 매핑 |
| [[unit-tests]] | 도메인 / aggregate |
| [[integration-tests]] | Testcontainers + EmbeddedKafka |
| [[pg-mock-tests]] | PG sandbox / WireMock |

---

## 2. 도구

| 도구 | 영역 |
| --- | --- |
| JUnit 5 | 단위 |
| Mockito | mock |
| AssertJ | assertions |
| Testcontainers | PostgreSQL + Redis + Kafka |
| EmbeddedKafka | Spring Kafka Test |
| WireMock | PG sandbox mock |
| RestAssured | API |
| Awaitility | 비동기 (worker / webhook) |

---

## 3. TDD 흐름

```
RED:   AC → 실패 테스트 작성
GREEN: 도메인 / service 구현
REFACTOR: 코드 정리 + 다른 테스트 통과 확인
```

---

## 4. 관련

- [[../product|↑ hub]]
- [[../requirements]] — AC
- [[../implementation/implementation]]
