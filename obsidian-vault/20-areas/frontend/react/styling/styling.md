---
title: "Styling in React — 스타일링 길잡이"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, styling, css, hub]
---

# Styling in React

**[[../react|↑ React Hub]]**

> 스타일링 방법은 여러 선택지. **선택은 팀 컨벤션**. 같은 프로젝트 안 1 개로 통일.

## 1. 선택지

| | 특징 | 예 |
| --- | --- | --- |
| **순수 CSS / CSS Modules** | 가장 단순. scope 분리. | `.module.css` |
| **styled-components** | JS 안 CSS. 컴포넌트 = 스타일. | `styled.div\`\`` |
| **Emotion** | styled-components 와 유사 + 더 가벼움. | `@emotion/react` |
| **Tailwind CSS** | utility class. 매우 빠른 prototyping. | `<div className="p-4 bg-blue-500">` |
| **CSS-in-JS (vanilla-extract)** | 빌드 타임 CSS-in-JS. zero-runtime. | TS 타입 안전 |
| **MUI / antd 등 UI lib 내장** | 라이브러리 안 styling 시스템. | sx prop |

## 2. masterway-dev 매핑

```bash
grep -E "styled-components|emotion|tailwind|antd|@mui" ~/masterway-dev/answer-fe/package.json
grep -E "styled-components|emotion|tailwind|antd|@mui" ~/masterway-dev/job-answer-fe/package.json
```

- `answer-fe`: styled-components + MUI.
- `job-answer-fe`: Tailwind + antd.
- `edu-masterway-fe`: 환경별 다름 (확인 필요).

→ 두 큰 갈래 다 다룸: [[styled-components]] / [[tailwind]].

## 3. 선택 가이드

```
프로토타이핑 빠름 + design system 자유 = Tailwind
컴포넌트 단위 캡슐화 + props 기반 동적 스타일 = styled-components / Emotion
완성된 UI 컴포넌트 + 그 스타일 사용 = MUI / antd (내장 styling)
간단 + scope = CSS Modules
```

`masterway-dev/answer-fe` (styled-components) 스타일:
```tsx
const Button = styled.button<{ $variant: 'primary' | 'ghost' }>`
  padding: 8px 16px;
  background: ${p => p.$variant === 'primary' ? '#000' : 'transparent'};
`;
```

`masterway-dev/job-answer-fe` (Tailwind) 스타일:
```tsx
<button className="px-4 py-2 bg-black text-white rounded hover:bg-gray-800">
  Click
</button>
```

## 4. 순수 CSS 의 빠른 시작

```tsx
// App.css
.title { color: red; font-size: 24px; }

// App.tsx
import './App.css';
<h1 className="title">Hello</h1>
```

→ 전역 충돌 위험. 큰 프로젝트 비추.

## 5. CSS Modules — scope 자동 분리

```tsx
// Button.module.css
.button { padding: 8px 16px; }

// Button.tsx
import styles from './Button.module.css';
<button className={styles.button}>Click</button>
```

→ 빌드 시 `styles.button` 이 unique 이름 (예: `Button_button__a1b2c`).
class 충돌 없음. **단순하고 안전**.

## 6. inline style — `style={{ ... }}`

```tsx
<div style={{ padding: 16, background: 'red', fontSize: '14px' }}>...</div>
```

- 동적 값에 OK.
- pseudo-class (`:hover`) / media query 불가.
- prop key 는 camelCase.
- 값은 number (px 자동) 또는 string.

→ 한두 곳에서만. 메인 스타일은 다른 방식.

## 7. className 동적 — clsx / classnames

```bash
yarn add clsx
```

```tsx
import clsx from 'clsx';

<button className={clsx(
  'btn',
  isPrimary && 'btn-primary',
  isDisabled && 'btn-disabled',
  size === 'lg' && 'btn-lg'
)}>
  Click
</button>
```

→ 조건부 class 의 표준.

## 8. CSS 전역 reset

```css
/* index.css 또는 styles/global.css */
* { box-sizing: border-box; margin: 0; padding: 0; }
body { font-family: 'Pretendard', sans-serif; }
```

→ vite 기본 reset 거의 없음. tailwind 는 자체 preflight 포함.

## 9. design token 시스템

```ts
// theme.ts
export const theme = {
  color: {
    primary: '#0064FF',
    text: '#1A1A1A',
    bg: '#FFFFFF',
  },
  space: { sm: 8, md: 16, lg: 24 },
  font: { sm: 12, md: 14, lg: 16 },
  radius: { sm: 4, md: 8, lg: 16 },
};

// styled-components
import { ThemeProvider } from 'styled-components';
<ThemeProvider theme={theme}>...</ThemeProvider>

// 사용
const Box = styled.div`
  padding: ${p => p.theme.space.md}px;
  color: ${p => p.theme.color.primary};
`;
```

→ Tailwind 도 `tailwind.config.js` 의 theme 으로 동일.

## 10. 학습 우선순위

1. **자기 프로젝트의 styling 방식 확인** (위 §2).
2. **그것 우선 학습** ([[styled-components]] 또는 [[tailwind]]).
3. **inline style + clsx + CSS Module 정도 익히기**.
4. **theme / design token** 설계.

## 11. 함정

1. **CSS 우선순위** — `!important` 남발 금지. 좋은 selector 설계.
2. **CSS in JS 런타임 비용** — styled-components 가 컴파일 시 X. 런타임 X 원하면 vanilla-extract.
3. **Tailwind class 가 길어짐** — `class="px-4 py-2 ..."` 30 개. 추출 (`@apply`) 또는 컴포넌트.
4. **글로벌 CSS + CSS Module 혼용** — 우선순위 헷갈림.
5. **반응형 / 다크모드** — 처음부터 설계. mobile-first.
6. **font / image asset 경로** — Vite 의 `import` 또는 `public/` 폴더 규칙.

## 12. 외부 자료

- [styled-components](https://styled-components.com/)
- [Tailwind CSS](https://tailwindcss.com/)
- [Pretendard](https://github.com/orioncactus/pretendard) (한글 무료 폰트)

## 13. 관련

- [[styled-components]]
- [[tailwind]]
- [[../ui-libraries/ui-libraries]]
