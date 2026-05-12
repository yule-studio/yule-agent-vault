---
title: "qa-engineer — Reference"
kind: knowledge
project: agent-references
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-12T18:30:00+09:00
tags:
  - reference
  - links
  - engineering
  - qa
---

# qa-engineer — Reference

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 15 링크 |

**[[engineering|↑ Engineering 인덱스]]**

## Reference 링크

- [Google Testing Blog](https://testing.googleblog.com/) — Google 의 테스트 패턴 / 안티패턴
- [xUnit Patterns (Gerard Meszaros)](http://xunitpatterns.com/) — 테스트 패턴의 표준 책 — 온라인 공개
- [Martin Fowler — Testing](https://martinfowler.com/tags/testing.html) — Test Pyramid / Mock / Contract test 정의
- [Pact 공식 docs](https://docs.pact.io/) — Consumer-driven contract test 표준
- [Property-based Testing 가이드 (Hypothesis)](https://hypothesis.readthedocs.io/en/latest/) — Python PBT 라이브러리
- [Mutation testing — Stryker](https://stryker-mutator.io/docs/) — 테스트 quality 검증 (테스트의 테스트)
- [Test Doubles guide (Mockito)](https://site.mockito.org/) — Java mock 표준
- [Playwright Test](https://playwright.dev/docs/intro) — e2e 표준 (frontend QA)
- [Cypress Best Practices](https://docs.cypress.io/guides/references/best-practices) — e2e 안티패턴
- [Snapshot Testing — Pros/Cons](https://kentcdodds.com/blog/effective-snapshot-testing) — 스냅샷 테스트 모범 사례
- [Test Containers](https://testcontainers.com/) — integration test 의 표준 (Docker 기반)
- [Atlassian — Test plan template](https://www.atlassian.com/agile/software-development/testing) — release blocker 정의
- [AAA pattern (Arrange-Act-Assert)](https://medium.com/@pjbgf/title-testing-code-ocd-and-the-aaa-pattern-df453975ab80) — 테스트 구조 표준
- [Coverage 의 함정](https://martinfowler.com/bliki/TestCoverage.html) — 커버리지 % 신뢰의 한계
- [SQLite 의 테스트 전략](https://www.sqlite.org/testing.html) — 미친 수준의 테스트 사례 (참고 영감)

## 운영 규칙

1. 링크 검증 — 죽은 링크는 발견 시 즉시 제거.
2. 새 링크 추가 시 한 줄 설명 (왜 좋은지) 필수.
3. 깊은 학습 자료는 `30-resources/` 에 별도 노트로 추출 + 본 카탈로그에는 링크만.