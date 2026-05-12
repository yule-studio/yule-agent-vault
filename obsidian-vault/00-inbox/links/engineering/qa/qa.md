---
title: "qa-engineer — Reference 카탈로그"
kind: knowledge
project: agent-references
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-12T21:30:00+09:00
tags:
  - reference
  - links
  - index
  - qa
---

# qa-engineer — Reference 카탈로그

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 12 카탈로그 |

**[[engineering|↑ Engineering]]**

> qa-engineer 의 책임 (테스트 전략 / contract / regression / 자동화) 영역별 카탈로그.

## 카탈로그

| 주제 | 진입점 | 한 줄 |
| --- | --- | --- |
| 테스트 전략 / 피라미드 | [[test-strategy]] | Pyramid / Trophy / Honeycomb 패턴 |
| 단위 테스트 | [[unit-testing]] | JUnit / pytest / Vitest / Jest |
| 통합 테스트 | [[integration-testing]] | Testcontainers / WireMock / DB seeding |
| E2E 테스트 | [[e2e-testing]] | Playwright / Cypress / Selenium |
| Contract 테스트 | [[contract-testing]] | Pact / consumer-driven |
| Property-based 테스트 | [[property-based-testing]] | QuickCheck / Hypothesis / jqwik |
| Mutation 테스트 | [[mutation-testing]] | Stryker / PIT — 테스트의 테스트 |
| 성능 / 부하 테스트 | [[performance-testing]] | k6 / JMeter / Gatling / Locust |
| 보안 테스트 | [[security-testing]] | OWASP ZAP / Burp Suite / SAST/DAST |
| 테스트 자동화 | [[test-automation]] | CI 통합 / flake 관리 / parallelization |
| BDD / TDD | [[bdd-tdd]] | Cucumber / Gherkin / Red-Green-Refactor |
| Release blocker / quality bar | [[release-blocker-criteria]] | 출시 결정 기준 |