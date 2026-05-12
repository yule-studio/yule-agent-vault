---
title: "Testing Library"
kind: knowledge
project: agent-references
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-12T20:30:00+09:00
tags:
  - reference
  - links
  - testing
  - testing-library
---

# Testing Library

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 13 링크 |

**[[testing-frontend|↑ Frontend 테스트]]**

> user-centric 컴포넌트 테스트. queryByRole / userEvent 가 핵심.

## Reference 링크

- [Testing Library 공식](https://testing-library.com/docs/) — 권위 docs
- [React Testing Library](https://testing-library.com/docs/react-testing-library/intro/) — React 통합
- [Vue Testing Library](https://testing-library.com/docs/vue-testing-library/intro/) — Vue 통합
- [user-event v14](https://testing-library.com/docs/user-event/intro/) — 실제 사용자 이벤트 시뮬레이션
- [Common mistakes (Kent Dodds)](https://kentcdodds.com/blog/common-mistakes-with-react-testing-library) — RTL 안티패턴
- [Testing Library Queries](https://testing-library.com/docs/queries/about) — getByRole / getByText 등
- [jest-dom matchers](https://github.com/testing-library/jest-dom) — toBeInTheDocument 등
- [Testing Implementation Details (Kent)](https://kentcdodds.com/blog/testing-implementation-details) — 구현 vs 행동 테스트
- [cheatsheet (RTL)](https://testing-library.com/docs/react-testing-library/cheatsheet/) — API 치트
- [How to test custom hooks](https://kentcdodds.com/blog/how-to-test-custom-react-hooks) — hook 테스트 패턴
- [Testing accessibility](https://testing-library.com/docs/queries/about/#priority) — a11y 우선 쿼리
- [Testing Library Source](https://github.com/testing-library) — 모노레포
- [Async queries (waitFor / findBy)](https://testing-library.com/docs/dom-testing-library/api-async/) — async 패턴