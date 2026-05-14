---
title: "React — Hub"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T04:00:00+09:00
tags: [area, frontend, react, hub, learning-path]
---

# React — Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | placeholder |
| v.2.0.0 | 2026-05-14 | engineering-agent/tech-lead | masterway-dev/-fe 프로젝트 스택 기준 학습 가이드 hub 로 재편 |

**[[../frontend|↑ frontend]]**

> **"프론트엔드 처음 배우는 사람을 위한 React 학습 hub."** 각 항목은 처음 보는 사람도 따라할 수 있는 단계별 노트 + 실 코드 + 함정 으로 구성. 실 프로젝트 (`masterway-dev/*-fe`) 의 스택 기준.

## 0. 이 hub 의 사용법

1. **순서대로 읽기** — 아래 §1 "Quick Start 로드맵" 부터.
2. **모르는 단어는 항상 옆 문서로 jump** — Obsidian wiki link (`[[...]]`).
3. **각 노트의 "코드 예시" 는 그대로 복사해서 돌려보기**.
4. **"함정" 섹션은 반드시 읽기** — 미리 알면 디버깅 시간 1/10.
5. **막히면 자기 프로젝트 (e.g. `answer-fe`, `uready-fe`) 에서 동일 패턴 검색** — 실 사례가 가장 좋은 학습.

## 1. Quick Start 로드맵 — 한 달 기준

### Week 1 — 기초 다지기
1. **무엇이 React 인가** — [[getting-started]]
2. **프로젝트 구조 이해** — [[project-structure]]
3. **개발 환경 설정** — [[configuration]]
4. **컴포넌트와 JSX** — [[core-concepts/components-and-jsx]]
5. **props 와 state** — [[core-concepts/props-and-state]]

### Week 2 — 핵심 hook
6. **필수 hook 5종** — [[core-concepts/hooks-essentials]] (useState/useEffect/useRef/useMemo/useCallback)
7. **조건부 / 리스트 렌더링** — [[core-concepts/conditional-and-list-rendering]]
8. **이벤트 처리** — [[core-concepts/events-and-synthetic]]
9. **폼 다루기** — [[core-concepts/forms-and-controlled-inputs]]
10. **TypeScript 기본** — [[typescript/typescript]]

### Week 3 — 실 프로젝트 패턴
11. **라우팅** — [[routing/routing]] + [[routing/react-router-v6]]
12. **상태 관리** — [[state-management/state-management]] (어느 것 골라야 하나)
13. **서버 데이터** — [[server-state/server-state]] + [[server-state/react-query]]
14. **API 호출** — [[http/http]] + [[http/axios-setup]]
15. **스타일링** — [[styling/styling]]

### Week 4 — 실무 패턴
16. **UI 라이브러리** — [[ui-libraries/ui-libraries]] (MUI / antd)
17. **인증 / 로그인** — [[auth/auth]] + [[auth/login-jwt]] + [[auth/oauth-social]]
18. **폼 라이브러리** — [[forms/forms]] + [[forms/react-hook-form]]
19. **성능 최적화** — [[performance/performance]]
20. **자주 하는 실수** — [[pitfalls/pitfalls]]

### 그 후
- [[recipes/recipes]] — modal / infinite scroll / drag-drop / pdf export
- [[testing/testing]] — 테스트
- [[../../computer-science/computer-science|computer-science]] — CS 기초

---

## 2. 주제 별 진입점

### 시작
- [[getting-started]] — 첫 React 프로젝트 만들기 (Vite + TypeScript)
- [[project-structure]] — `answer-fe` 같은 실 프로젝트의 폴더 구조
- [[configuration]] — vite.config.ts / tsconfig.json / eslint / prettier

### Core
- [[core-concepts/core-concepts]] — Core 개념 hub
- [[core-concepts/components-and-jsx]] — 컴포넌트 = 함수
- [[core-concepts/props-and-state]] — 데이터 전달과 상태
- [[core-concepts/hooks-essentials]] — useState / useEffect / useRef / useMemo / useCallback
- [[core-concepts/conditional-and-list-rendering]] — `&&`, `?:`, `map`
- [[core-concepts/events-and-synthetic]] — onClick / onChange / synthetic event
- [[core-concepts/forms-and-controlled-inputs]] — controlled vs uncontrolled

### TypeScript
- [[typescript/typescript]] — TypeScript hub
- [[typescript/component-types]] — 컴포넌트 / props 타입 정의

### Routing
- [[routing/routing]] — Routing hub
- [[routing/react-router-v6]] — react-router-dom v6 라우팅 표준

### 상태 관리
- [[state-management/state-management]] — State 개념 hub (local vs server vs global)
- [[state-management/jotai]] — Jotai (atomic state, `answer-fe` 사용)
- [[state-management/recoil]] — Recoil (Facebook, `answer-fe` 사용)
- [[state-management/zustand]] — Zustand (간단한 store, 모던 표준)
- [[state-management/state-strategy]] — 어떤 상태 관리 라이브러리를 골라야 하나

### 서버 데이터
- [[server-state/server-state]] — Server state vs client state
- [[server-state/react-query]] — TanStack Query (v3 vs v4 vs v5)

### HTTP
- [[http/http]] — HTTP hub
- [[http/axios-setup]] — axios instance + baseURL + timeout
- [[http/interceptors]] — request / response interceptor + token refresh

### 스타일링
- [[styling/styling]] — Styling hub
- [[styling/styled-components]] — CSS-in-JS (`answer-fe` 사용)
- [[styling/tailwind]] — Tailwind CSS (`job-answer-fe` 사용)

### UI 라이브러리
- [[ui-libraries/ui-libraries]] — UI libs hub
- [[ui-libraries/mui-material]] — Material UI (`answer-fe` 사용)
- [[ui-libraries/antd]] — Ant Design (`answer-fe`/`job-answer-fe` 사용)

### 인증
- [[auth/auth]] — 인증 hub
- [[auth/login-jwt]] — JWT + localStorage / cookie
- [[auth/oauth-social]] — Google / Apple / Kakao 소셜 로그인 (실 프로젝트 패턴)

### 폼
- [[forms/forms]] — 폼 hub
- [[forms/react-hook-form]] — react-hook-form + zod validation

### 성능
- [[performance/performance]] — Performance hub
- [[performance/memoization]] — React.memo / useMemo / useCallback
- [[performance/code-splitting]] — `lazy()` + `Suspense` + dynamic import
- [[performance/virtual-list]] — react-window (`answer-fe` 사용)

### 함정 (실수 사전)
- [[pitfalls/pitfalls]] — Pitfalls hub
- [[pitfalls/re-render-loops]] — 무한 re-render
- [[pitfalls/stale-closures]] — closure 안 옛 값

### 실 레시피 (실 프로젝트 응용)
- [[recipes/recipes]] — Recipes hub
- [[recipes/modal-portal]] — Modal + Portal
- [[recipes/infinite-scroll]] — 무한 스크롤

---

## 3. 학습 마음가짐

1. **모든 걸 한 번에 외우려 하지 마.** 자주 보는 5-10 개부터.
2. **공식 docs 가 거의 항상 답.** `react.dev`, MDN, 각 라이브러리 docs.
3. **에러 메시지를 영어로 그대로 구글.** Stack Overflow / GitHub Issues 답이 옴.
4. **자기 프로젝트에서 동일 패턴 찾기.** `grep -r "useState" src/` 가 가장 빠른 학습.
5. **AI 에게 물어볼 때**: "왜" 와 "언제" 를 묻기. "How to" 만 묻지 말 것.

---

## 4. 실 프로젝트 매핑

`masterway-dev/` 의 각 `-fe` 프로젝트와 본 hub 의 연결:

| 프로젝트 | 사용 스택 | 본 hub 의 참고 노트 |
| --- | --- | --- |
| **answer-fe** | React 18, MUI, antd, jotai, recoil, react-query v4, styled-components, react-window | 거의 모든 노트 |
| **edu-masterway-fe** | React 18, MUI, antd, react-query v4 | core / routing / server-state / styling |
| **job-answer-fe** | React 18, antd, Tailwind, react-query v3, OAuth (Google/Apple/Kakao) | tailwind / oauth-social |
| **uready-fe** | React 19 (minimal), 학습용 / 신규 | getting-started 그대로 |
| **masterway-html-fe** | HTML 정적 / 디자인 시안 보존 | 디자인 reference |

---

## 5. 외부 학습 자료

### 공식
- **react.dev** — https://react.dev (한글 docs 있음)
- **TypeScript** — https://www.typescriptlang.org/docs/
- **TanStack Query** — https://tanstack.com/query/latest
- **react-router** — https://reactrouter.com
- **MUI** — https://mui.com
- **Ant Design** — https://ant.design

### 한국어 강의 / 책
- **인프런** "한 입 크기로 잘라먹는 리액트" (이정환)
- **인프런** "Slack 클론 코딩" (제로초)
- **모던 리액트 Deep Dive** (김용찬)
- **Vlpt Velopert** — https://react.vlpt.us (무료, 한국어)

### 영문 추천
- **Kent C. Dodds** — https://kentcdodds.com/blog
- **Dan Abramov** — https://overreacted.io (React 핵심 개발자)
- **Josh Comeau** — https://www.joshwcomeau.com (시각적 설명 최고)
- **TKDodo** — https://tkdodo.eu/blog (react-query 깊은 분석)

---

## 6. 관련

- [[../frontend|↑ frontend]]
- [[../typescript/typescript|↗ TypeScript hub]]
- [[../html-css/html-css|↗ HTML/CSS hub]]
- [[../testing-frameworks/testing-frameworks|↗ Testing hub]]
- [[../../computer-science/computer-science|↗ CS hub]]
