---
title: "frontend — 프론트엔드 영역 Hub"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-19T00:40:00+09:00
tags: [area, frontend, hub]
home_hub: areas
related:
  - "[[../areas]]"
  - "[[_common/_common]]"
  - "[[react/react]]"
  - "[[vue/vue]]"
  - "[[nextjs/nextjs]]"
  - "[[../computer-science/network/http/http]]"
---

# frontend — 프론트엔드 영역 Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-19 | engineering-agent/tech-lead | 최초 — 17 영역 통합 hub |

**[[../areas|↑ 20-areas]]**

---

## 1. 영역 정의

본 영역은 브라우저 / 모바일 웹 / 데스크탑 웹 응용을 만드는 데 사용되는 **언어 / 프레임워크 / 빌드 / 테스트 / 디자인 시스템 / 성능 최적화** 의 인덱스다.

본 영역이 정의하는 것:
- 17 영역의 진입점
- 영역 간 책임 분리 (언어 / 프레임워크 / 빌드 / 상태 관리 / 성능 / 디자인 시스템 / 테스트)
- 학습 진입점 / 권장 순서
- 인접 영역 (HTTP / 디자인 / 백엔드) 과의 cross-link

본 영역이 정의하지 않는 것:
- 백엔드 API — [[../backend/backend]]
- HTTP 프로토콜 자체 — [[../computer-science/network/http/http]]
- 디자인 결정 / UX — [[../design/]]

---

## 2. 영역 분류

### 2.1 언어 / 표준

| 디렉토리 | 진입점 | 다루는 주제 |
| --- | --- | --- |
| [[html-css/|html-css]] | (hub 필요) | 시멘틱 HTML / CSS / 접근성 |
| [[javascript/|javascript]] | (hub 필요) | ES6+ / 모듈 / async / 표준 API |
| [[typescript/|typescript]] | (hub 필요) | 타입 시스템 / 제네릭 / 타입 안전 |

### 2.2 메이저 프레임워크 / 메타 프레임워크

| 디렉토리 | 진입점 | 비고 |
| --- | --- | --- |
| [[react/react|react]] | react.md | 가장 큰 ecosystem |
| [[vue/vue|vue]] | vue.md | Composition API + reactivity |
| [[angular/angular|angular]] | angular.md | Google / RxJS 통합 |
| [[svelte-sveltekit/svelte-sveltekit|svelte-sveltekit]] | svelte-sveltekit.md | compile-time framework |
| [[solidjs/solidjs|solidjs]] | solidjs.md | fine-grained reactivity |
| [[nextjs/nextjs|nextjs]] | nextjs.md | React 메타 프레임워크 (App Router) |
| [[nuxt/nuxt|nuxt]] | nuxt.md | Vue 메타 프레임워크 |

### 2.3 빌드 / 도구

| 디렉토리 | 진입점 |
| --- | --- |
| [[build-tools/build-tools|build-tools]] | Vite / Webpack / Rollup / esbuild / Turbopack |
| [[testing-frameworks/testing-frameworks|testing-frameworks]] | Vitest / Jest / Playwright / Cypress |

### 2.4 상태 / UI / 디자인

| 디렉토리 | 진입점 |
| --- | --- |
| [[state-management/state-management|state-management]] | Redux / Zustand / Jotai / Recoil / Pinia / Signals |
| [[ui-libraries/ui-libraries|ui-libraries]] | shadcn / MUI / Chakra / Mantine / Ant Design |
| [[design-systems/design-systems|design-systems]] | 토큰 / 컴포넌트 / 문서화 / Figma 연동 |

### 2.5 공통 / 성능

| 디렉토리 | 진입점 |
| --- | --- |
| [[_common/_common|_common]] | 모든 프레임워크 공통 패턴 |
| [[performance-optimization/performance-optimization|performance-optimization]] | Core Web Vitals / 번들 / 렌더링 / 캐시 |

---

## 3. 학습 진입점 권장

| 목적 | 진입점 |
| --- | --- |
| 프론트엔드 입문 | [[html-css/|html-css]] → [[javascript/|javascript]] → [[typescript/|typescript]] → 1 프레임워크 선택 |
| React 시작 | [[react/react]] → [[state-management/state-management]] → [[testing-frameworks/testing-frameworks]] |
| Vue 시작 | [[vue/vue]] → [[state-management/state-management]] |
| SSR / 풀스택 | [[nextjs/nextjs]] (React) / [[nuxt/nuxt]] (Vue) |
| 성능 최적화 | [[performance-optimization/performance-optimization]] → [[build-tools/build-tools]] |
| 디자인 / UI 통일 | [[design-systems/design-systems]] → [[ui-libraries/ui-libraries]] |
| 테스트 | [[testing-frameworks/testing-frameworks]] → 각 프레임워크의 testing 노트 |

---

## 4. 프레임워크 선택 결정 매트릭스

| 조건 | 권장 |
| --- | --- |
| 가장 큰 ecosystem / 채용 시장 | React + Next.js |
| 학습 곡선 낮음 + 작은 팀 | Vue + Nuxt |
| 기업 표준 + DI / RxJS | Angular |
| 번들 크기 / 성능 최우선 | Svelte / Solid |
| 정적 사이트 / 컨텐츠 | Astro / Hugo (별도) |
| 다중 프레임워크 host | Web Components / Module Federation |
| SSR / Edge / Streaming | Next.js / Nuxt / SvelteKit / Remix |
| 100% 클라이언트 SPA | React / Vue / Solid + Vite |

---

## 5. 빌드 / 테스트 선택

| 빌드 | 적합 |
| --- | --- |
| Vite | SPA / 라이브러리 — 대부분의 신규 |
| Webpack | 기존 / 복잡한 설정 |
| esbuild / Rollup | 라이브러리 / 빠른 빌드 |
| Turbopack | Next.js 기본 (점진 적용) |

| 테스트 | 적합 |
| --- | --- |
| Vitest | Vite 기반 — 신규 표준 |
| Jest | React Native / 기존 |
| Playwright | E2E (모든 브라우저) |
| Cypress | E2E (React/Vue 중심) |
| Testing Library | unit + integration (Jest/Vitest 보조) |

---

## 6. 인접 영역과의 책임 분리

| 본 영역 | 인접 영역 | 분리 기준 |
| --- | --- | --- |
| 프레임워크 사용법 | TypeScript 언어 자체 → [[typescript/]] | 적용 vs 언어 |
| 컴포넌트 / 상태 | HTTP API 통신 → [[../computer-science/network/http/http]] | 적용 vs 프로토콜 |
| 디자인 시스템 | UX / 디자인 결정 → [[../design/]] | 구현 vs 결정 |
| 빌드 / 배포 | 인프라 / 호스팅 → [[../devops/devops]] | 도구 vs 인프라 |
| 성능 최적화 | 네트워크 / CDN → [[../computer-science/network/network]] | 클라이언트 vs 네트워크 |

---

## 7. 공통 결정 — 새 프론트엔드 프로젝트 시작

| 결정 | 옵션 |
| --- | --- |
| 프레임워크 | §4 결정 매트릭스 |
| 빌드 도구 | §5 Vite (기본) / Next/Nuxt 자체 빌드 |
| 언어 | TypeScript (기본) / JavaScript (legacy) |
| 상태 관리 | server state (TanStack Query) + client state (Zustand / Pinia) 분리 |
| UI | shadcn (custom) / MUI / Mantine / 자체 |
| 디자인 시스템 | 토큰 + 컴포넌트 라이브러리 분리 |
| 테스트 | Vitest + Testing Library + Playwright (E2E) |
| 패키지 매니저 | pnpm (기본) / npm / yarn |

---

## 8. 본 영역의 유지보수

| 트리거 | 갱신 항목 |
| --- | --- |
| 새 프레임워크 / 도구 등장 | §2 분류 + 신규 디렉토리 + hub |
| 결정 매트릭스 변경 | §4, §5 |
| 인접 영역 책임 변경 | §6 |
| 누락된 sub-hub 추가 (html-css / javascript / typescript) | §2.1 |

---

## 9. 관련

- [[../areas|↑ 20-areas]]
- [[_common/_common]]
- [[react/react]]
- [[vue/vue]]
- [[nextjs/nextjs]]
- [[nuxt/nuxt]]
- [[performance-optimization/performance-optimization]]
- [[build-tools/build-tools]]
- [[testing-frameworks/testing-frameworks]]
- [[design-systems/design-systems]]
- [[../computer-science/network/http/http|↗ HTTP hub]]
- [[../backend/backend|↗ backend hub]]
- [[../devops/devops|↗ devops hub]]
