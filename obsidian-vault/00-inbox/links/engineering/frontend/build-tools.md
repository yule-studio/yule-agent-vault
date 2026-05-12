---
title: "빌드 도구 — Vite / Webpack / Turbopack"
kind: knowledge
project: agent-references
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-12T20:30:00+09:00
tags:
  - reference
  - links
  - frontend
  - build-tools
---

# 빌드 도구 — Vite / Webpack / Turbopack

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 15 링크 |

**[[frontend|↑ Frontend]]**

> Modern 번들러 + 빌드 시스템.

## Reference 링크

- [Vite 공식 docs](https://vitejs.dev/guide/) — Modern 번들러 표준
- [Webpack 공식 docs](https://webpack.js.org/concepts/) — 레거시이지만 여전히 많음
- [Turbopack](https://turbo.build/pack/docs) — Next.js 의 Rust 기반 번들러
- [esbuild](https://esbuild.github.io/) — 초고속 번들러 (Vite 내부)
- [SWC](https://swc.rs/) — Rust 기반 transpiler (Next.js 내부)
- [Rollup](https://rollupjs.org/) — 라이브러리용 번들러 (Vite 내부)
- [Parcel](https://parceljs.org/) — zero-config 번들러
- [Babel docs](https://babeljs.io/docs/) — JS transpiler 표준
- [npm vs pnpm vs yarn](https://pnpm.io/motivation) — pnpm 의 동기
- [Bun (런타임 + 번들러)](https://bun.sh/docs) — Node.js 대안
- [Tree shaking 가이드](https://webpack.js.org/guides/tree-shaking/) — 미사용 코드 제거
- [Code splitting (Webpack)](https://webpack.js.org/guides/code-splitting/) — 청크 분리
- [Bundle size analyzer](https://github.com/webpack-contrib/webpack-bundle-analyzer) — webpack 번들 분석
- [Vite plugins](https://vitejs.dev/plugins/) — 공식 + 커뮤니티 플러그인
- [Module Federation](https://webpack.js.org/concepts/module-federation/) — 마이크로 프론트엔드