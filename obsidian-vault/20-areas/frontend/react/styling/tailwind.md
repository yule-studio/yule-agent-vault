---
title: "Tailwind CSS — utility class 기반"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, styling, tailwind, utility]
---

# Tailwind CSS

**[[styling|↑ Styling Hub]]**

> `masterway-dev/job-answer-fe` 표준. JSX 의 className 으로 모든 스타일.

## 1. 설치 (Vite + Tailwind v3)

```bash
yarn add -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

```ts
// tailwind.config.js
export default {
  content: ['./index.html', './src/**/*.{ts,tsx}'],
  theme: { extend: {} },
  plugins: [],
};
```

```css
/* src/index.css */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

```ts
// main.tsx
import './index.css';
```

→ Tailwind v4 (2024+) 는 `@import "tailwindcss"` 1 줄로 단순화.

## 2. 가장 작은 예

```tsx
<button className="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600">
  Click
</button>
```

→ class 하나하나 = utility.

## 3. 자주 쓰는 utility 분류

### spacing — padding / margin
```
p-4    padding: 1rem
px-4   padding-left/right: 1rem
py-2   padding-top/bottom: 0.5rem
pt-2   padding-top: 0.5rem
m-4    margin: 1rem
mx-auto margin-left/right: auto (center)
```

### sizing
```
w-full   width: 100%
w-1/2    width: 50%
w-screen width: 100vw
h-screen height: 100vh
max-w-md max-width: 28rem
```

### flex / grid
```
flex                  display: flex
flex-row / flex-col
items-center          align-items: center
justify-center        justify-content: center
gap-4                 gap: 1rem

grid                  display: grid
grid-cols-3
```

### color
```
text-blue-500
bg-red-500
border-gray-300
```

### text
```
text-sm / text-lg / text-2xl
font-bold / font-medium
text-center / text-left
leading-relaxed
```

### border / radius / shadow
```
border / border-2
rounded / rounded-lg / rounded-full
shadow / shadow-lg
```

### state
```
hover:bg-blue-600
focus:ring-2
disabled:opacity-50
group-hover:scale-105
```

### responsive (mobile-first)
```
sm:px-4    640px+
md:px-6    768px+
lg:px-8    1024px+
xl:px-12   1280px+
```

```tsx
<div className="px-2 md:px-6 lg:px-12">
  {/* mobile: 8px, tablet: 24px, desktop: 48px */}
</div>
```

## 4. dark mode

```ts
// tailwind.config.js
darkMode: 'class',  // <html class="dark"> 일 때
```

```tsx
<div className="bg-white text-black dark:bg-gray-900 dark:text-white">
  ...
</div>
```

```tsx
// toggle
document.documentElement.classList.toggle('dark');
```

## 5. arbitrary value — `[]`

```tsx
<div className="w-[37px] text-[#ff5733]">
  {/* preset 에 없는 값도 */}
</div>
```

→ 디자인 시스템 벗어나는 값. 가능한 적게.

## 6. 동적 class — clsx + tailwind-merge

```bash
yarn add clsx tailwind-merge
```

```ts
// lib/cn.ts
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

```tsx
import { cn } from '@/lib/cn';

<button className={cn(
  'px-4 py-2 rounded',
  variant === 'primary' && 'bg-blue-500 text-white',
  variant === 'ghost' && 'bg-transparent text-black',
  className,                  // 외부 override
)}>
  Click
</button>
```

→ `tailwind-merge` 가 충돌 class 자동 정리 (`p-4 p-2` → `p-2` 만).

## 7. 컴포넌트 추출

```tsx
// 같은 utility 조합 반복 → 컴포넌트
const Button = ({ variant = 'primary', className, ...rest }) => (
  <button
    className={cn(
      'px-4 py-2 rounded font-medium',
      variant === 'primary' && 'bg-blue-500 text-white hover:bg-blue-600',
      variant === 'ghost' && 'bg-transparent text-black hover:bg-gray-100',
      className,
    )}
    {...rest}
  />
);
```

→ 또는 `@apply` 디렉티브:
```css
.btn-primary {
  @apply px-4 py-2 rounded bg-blue-500 text-white hover:bg-blue-600;
}
```

→ `@apply` 는 신중히. 컴포넌트 추출이 보통 더 명확.

## 8. theme 커스터마이즈

```ts
// tailwind.config.js
export default {
  content: ['./src/**/*.{ts,tsx}'],
  theme: {
    extend: {
      colors: {
        primary: '#0064FF',
        secondary: '#FF6B00',
      },
      fontFamily: {
        sans: ['Pretendard', 'sans-serif'],
      },
      spacing: {
        '128': '32rem',
      },
    },
  },
};
```

```tsx
<div className="bg-primary text-secondary p-128">...</div>
```

## 9. forms / typography 플러그인

```bash
yarn add -D @tailwindcss/forms @tailwindcss/typography
```

```ts
plugins: [require('@tailwindcss/forms'), require('@tailwindcss/typography')],
```

- `forms` — input / select 의 기본 스타일.
- `typography` — `<article class="prose">` 로 마크다운 풍 글.

## 10. 자주 쓰는 패턴

### Centering
```tsx
<div className="flex items-center justify-center min-h-screen">
  <div>Centered</div>
</div>
```

### Card
```tsx
<div className="rounded-lg border border-gray-200 bg-white p-6 shadow-sm">
  ...
</div>
```

### Modal backdrop
```tsx
<div className="fixed inset-0 bg-black/50 flex items-center justify-center z-50">
  <div className="bg-white rounded-lg p-6 max-w-md w-full">...</div>
</div>
```

### Responsive nav
```tsx
<nav className="flex flex-col md:flex-row gap-2 md:gap-6">...</nav>
```

## 11. job-answer-fe 패턴

```bash
grep -rn "className=" ~/masterway-dev/job-answer-fe/src --include="*.tsx" | head -5
```

전형:
```tsx
<div className="flex flex-col gap-4 p-6 max-w-3xl mx-auto">
  <h1 className="text-2xl font-bold">제목</h1>
  <p className="text-gray-600">내용</p>
  <button className="px-4 py-2 bg-blue-500 text-white rounded">
    확인
  </button>
</div>
```

## 12. VS Code 설정

```json
// .vscode/settings.json
{
  "editor.quickSuggestions": { "strings": "on" },
  "tailwindCSS.experimental.classRegex": [
    ["cn\\(([^)]*)\\)", "[\"'`]([^\"'`]*).*?[\"'`]"]
  ]
}
```

→ tailwindcss extension 설치하면 자동 완성.

## 13. 함정

1. **content 경로 누락** — `tailwind.config.js` 의 content 가 src 못 잡으면 class 가 build 에 안 들어감.
2. **dynamic class string** — `bg-${color}-500` ❌. Tailwind 가 빌드 시 사용 class 분석 못함. 전체 class string 작성.
3. **`@apply` 남발** — Tailwind 의 목적 (utility) 약해짐.
4. **충돌 class** — `p-4 p-2` 의 결과 우연한 우선순위. `tailwind-merge` 필수.
5. **너무 긴 className** — 50 줄 넘으면 component 추출.
6. **prose 클래스 + 자체 스타일** — `prose` 가 child element 자동 스타일. 자체 컴포넌트와 충돌 가능.
7. **CSS Module 과 혼용** — 우선순위 헷갈림. 한 가지만.

## 14. 외부 자료

- [Tailwind CSS 공식](https://tailwindcss.com/)
- [Tailwind UI](https://tailwindui.com/) (유료 컴포넌트 모음)
- [shadcn/ui](https://ui.shadcn.com/) — Tailwind 기반 무료 컴포넌트.

## 15. 관련

- [[styling]]
- [[styled-components]]
- [[../ui-libraries/ui-libraries]]
