---
title: "MSW (Mock Service Worker)"
kind: knowledge
project: agent-references
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-12T20:30:00+09:00
tags:
  - reference
  - links
  - testing
  - msw
---

# MSW (Mock Service Worker)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 11 링크 |

**[[testing-frontend|↑ Frontend 테스트]]**

> API mocking 의 표준. fetch / XHR 을 service worker 로 가로챔.

## Reference 링크

- [MSW 공식 docs](https://mswjs.io/docs/) — 권위 docs
- [MSW with React](https://mswjs.io/docs/integrations/browser) — 브라우저 통합
- [MSW with Node (Jest/Vitest)](https://mswjs.io/docs/integrations/node) — Node 통합
- [MSW Source](https://github.com/mswjs/msw) — 소스 + 이슈
- [MSW Data (in-memory DB)](https://github.com/mswjs/data) — 데이터 모델링
- [MSW + Storybook](https://storybook.js.org/addons/msw-storybook-addon) — Storybook 통합
- [MSW Examples](https://github.com/mswjs/examples) — 공식 예시 모음
- [MSW vs nock vs miragejs](https://mswjs.io/docs/comparison) — 공식 비교
- [MSW with GraphQL](https://mswjs.io/docs/api/graphql) — GraphQL mocking
- [Kent Dodds — MSW intro](https://kentcdodds.com/blog/stop-mocking-fetch) — 왜 MSW 인가
- [REST handlers](https://mswjs.io/docs/api/http) — REST API mock