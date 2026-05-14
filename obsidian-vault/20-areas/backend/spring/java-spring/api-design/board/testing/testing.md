---
title: "board §11 — 테스트 (Hub)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T14:50:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - testing
---

# board §11 — 테스트 (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[../board|↑ hub]]**  ·  ← [[../transactions]]  ·  → [[../operations/operations]]

> Test Pyramid. signup 의 testing 패턴 + board 특화 시나리오.

---

## 1. 노트

| 노트 | 무엇 |
| --- | --- |
| [[test-scenarios]] | 시나리오 표 (8 영역 × 45 AC) |
| [[unit-tests]] | 도메인 / VO / Service 단위 |
| [[integration-tests]] | Testcontainers + counter race + AFTER_COMMIT |

→ [[../../signup/testing/testing|↗ signup testing]] 의 Test Pyramid / 환경 / 도구 그대로 적용.

---

## 2. board 특화 시나리오 (signup 과 다른 부분)

- **Counter race** — 동시 좋아요 / 댓글 count.
- **Comment tree** — N+1 회피 검증.
- **Cursor pagination** — 새 글 추가 시 boundary 정확도.
- **차단 filter** — 차단 후 즉시 반영.
- **모더 자동 hide** — 5회 임계값 정확도.
- **첨부 매핑** — S3 mock + post 연결.

자세히: [[test-scenarios]].

---

## 3. 도구

| 도구 | 사용 |
| --- | --- |
| JUnit 5 + AssertJ + Mockito | Unit |
| Testcontainers PostgreSQL 16 + Redis 7 | Integration |
| LocalStack S3 | S3 mock |
| WireMock | FCM mock |
| REST Assured | E2E |

---

## 4. 관련

- [[../board|↑ hub]]
- [[../../signup/testing/testing|↗ signup testing]] — 패턴
- [[test-scenarios]] · [[unit-tests]] · [[integration-tests]]
