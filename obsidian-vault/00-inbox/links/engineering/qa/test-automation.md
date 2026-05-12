---
title: "테스트 자동화 / CI 통합"
kind: knowledge
project: agent-references
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-12T21:30:00+09:00
tags:
  - reference
  - links
  - qa
  - test-automation
---

# 테스트 자동화 / CI 통합

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 15 링크 |

**[[qa|↑ QA]]**

> CI/CD 안의 테스트 / parallel / flake 관리 / coverage 게이트.

## Reference 링크

- [GitHub Actions matrix](https://docs.github.com/en/actions/using-jobs/using-a-matrix-for-your-jobs) — 병렬 테스트
- [Test parallelization (Cypress)](https://docs.cypress.io/guides/cloud/smart-orchestration/parallelization) — 병렬 e2e
- [Codecov](https://docs.codecov.com/) — coverage 게이트
- [Coveralls](https://docs.coveralls.io/) — coverage alternative
- [Jacoco (Java)](https://www.jacoco.org/jacoco/trunk/doc/) — Java coverage
- [Istanbul / nyc (JS)](https://istanbul.js.org/) — JS coverage
- [Coverage.py (Python)](https://coverage.readthedocs.io/) — Python coverage
- [Flaky test detection (Google)](https://testing.googleblog.com/2016/05/flaky-tests-at-google-and-how-we.html) — flake 분석
- [Test impact analysis](https://docs.gitlab.com/ee/ci/pipelines/parent_child_pipelines.html) — 변경 영향 테스트
- [Test sharding](https://docs.gradle.org/current/userguide/performance.html#parallel_execution) — Gradle 병렬
- [Retry strategies](https://playwright.dev/docs/test-retries) — 재시도 정책
- [Test report (Allure)](https://allurereport.org/docs/) — Allure 보고서
- [Awesome Test Automation](https://github.com/atinfo/awesome-test-automation) — 자료 인덱스
- [GitLab CI test reports](https://docs.gitlab.com/ee/ci/testing/unit_test_reports.html) — CI 통합 보고서
- [Test Pyramid in CI](https://martinfowler.com/articles/practical-test-pyramid.html#TestsInTheDeploymentPipeline) — 파이프라인 테스트 배치