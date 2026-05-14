---
title: "styled-components — 컴포넌트 = 스타일"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, styling, styled-components, css-in-js]
---

# styled-components

**[[styling|↑ Styling Hub]]**

> `masterway-dev/answer-fe` 표준. JS 안 CSS, **컴포넌트 = 스타일**.

## 1. 설치

```bash
yarn add styled-components
yarn add -D @types/styled-components
```

## 2. 가장 작은 예

```tsx
import styled from 'styled-components';

const Title = styled.h1`
  color: red;
  font-size: 24px;
`;

function App() {
  return <Title>Hello</Title>;
}
```

→ `styled.태그명` + tagged template literal. 일반 컴포넌트처럼 사용.

## 3. props 기반 동적 스타일

```tsx
interface ButtonProps {
  $primary?: boolean;       // $ prefix = transient prop (DOM 으로 안 내려감)
  $size?: 'sm' | 'md' | 'lg';
}

const Button = styled.button<ButtonProps>`
  padding: ${p => p.$size === 'sm' ? '4px 8px' : '8px 16px'};
  background: ${p => p.$primary ? '#000' : 'transparent'};
  color: ${p => p.$primary ? '#fff' : '#000'};
  border: ${p => p.$primary ? 'none' : '1px solid #000'};

  &:hover {
    opacity: 0.8;
  }
`;

<Button $primary $size="md">Click</Button>
```

→ `$primary` 의 `$` 가 styled-components 만의 transient prop. DOM 의 `<button primary>` 가 아니라 `<button>` 으로 렌더.

## 4. 기존 컴포넌트 wrap

```tsx
// 기존 button 스타일 + 추가
const RedButton = styled.button`
  color: red;
`;

// 기존 RedButton + bold
const BoldRedButton = styled(RedButton)`
  font-weight: bold;
`;

// 기존 lib 컴포넌트 wrap (Link 등)
import { Link } from 'react-router-dom';
const StyledLink = styled(Link)`
  color: blue;
  text-decoration: none;
`;
```

## 5. as prop — 다른 element 로 렌더

```tsx
const Box = styled.div`
  padding: 16px;
`;

<Box>div 로 렌더</Box>
<Box as="section">section 으로 렌더</Box>
<Box as="a" href="...">a 태그로</Box>
```

## 6. global style

```tsx
import { createGlobalStyle } from 'styled-components';

const GlobalStyle = createGlobalStyle`
  * { box-sizing: border-box; margin: 0; padding: 0; }
  body { font-family: 'Pretendard', sans-serif; }
`;

function App() {
  return (
    <>
      <GlobalStyle />
      <Router>...</Router>
    </>
  );
}
```

## 7. theme — ThemeProvider

```tsx
import { ThemeProvider } from 'styled-components';

const theme = {
  color: { primary: '#0064FF', text: '#1A1A1A' },
  space: { sm: 8, md: 16, lg: 24 },
};

<ThemeProvider theme={theme}>
  <App />
</ThemeProvider>

// 사용
const Box = styled.div`
  padding: ${p => p.theme.space.md}px;
  color: ${p => p.theme.color.primary};
`;
```

### TypeScript theme 타입화

```ts
// styled.d.ts
import 'styled-components';

declare module 'styled-components' {
  export interface DefaultTheme {
    color: { primary: string; text: string };
    space: { sm: number; md: number; lg: number };
  }
}
```

→ `p.theme.color.primary` 자동 완성 + 타입 검사.

## 8. media query

```tsx
const Card = styled.div`
  padding: 16px;

  @media (min-width: 768px) {
    padding: 24px;
  }
`;

// 또는 theme 에 breakpoint
const Card = styled.div`
  padding: 16px;
  ${p => p.theme.media.tablet} {
    padding: 24px;
  }
`;
```

```ts
// theme
media: {
  mobile: '@media (max-width: 767px)',
  tablet: '@media (min-width: 768px)',
  desktop: '@media (min-width: 1024px)',
}
```

## 9. animation

```tsx
import styled, { keyframes } from 'styled-components';

const fadeIn = keyframes`
  from { opacity: 0; }
  to { opacity: 1; }
`;

const Box = styled.div`
  animation: ${fadeIn} 0.3s ease-in;
`;
```

## 10. css helper — 조각 재사용

```tsx
import { css } from 'styled-components';

const flexCenter = css`
  display: flex;
  justify-content: center;
  align-items: center;
`;

const Box = styled.div`
  ${flexCenter}
  padding: 16px;
`;

// 또는 props 기반
const Button = styled.button<{ $variant?: 'primary' | 'ghost' }>`
  padding: 8px 16px;
  ${p => p.$variant === 'primary' && css`
    background: black;
    color: white;
  `}
`;
```

## 11. answer-fe 의 흔한 폴더 구조

```
components/atom/
  Button/
    Button.tsx              // 컴포넌트 로직
    Button.styled.ts        // styled 컴포넌트들
    index.ts                // re-export
```

```tsx
// Button.styled.ts
import styled from 'styled-components';

export const Wrapper = styled.button<{ $primary?: boolean }>`
  padding: 8px 16px;
  background: ${p => p.$primary ? '#000' : '#fff'};
`;

export const Icon = styled.span`...`;

// Button.tsx
import * as S from './Button.styled';

export const Button = ({ primary, children }) => (
  <S.Wrapper $primary={primary}>{children}</S.Wrapper>
);
```

→ `S.Wrapper`, `S.Icon` 패턴이 매우 흔함.

## 12. 함정

1. **컴포넌트 안 styled 정의** — 매 render 마다 새 컴포넌트 = re-mount = state 손실 + 성능 폭주.
   ```tsx
   // ❌
   function Comp() {
     const Title = styled.h1`...`;    // 매 render 새로
     return <Title>...</Title>;
   }
   // ✅ 컴포넌트 밖
   const Title = styled.h1`...`;
   function Comp() { return <Title>...</Title>; }
   ```
2. **prop 의 `$` 누락** — `<Button primary>` 가 DOM 에 `<button primary>` 로 나옴 → React 경고.
3. **theme prop 의 typo** — `p.theme.colors.primary` vs `p.theme.color.primary`. TS 타입화로 방지.
4. **`!important` 남발** — selector 우선순위 풀어야.
5. **runtime CSS 비용** — 매우 큰 list 에 styled 컴포넌트 N 개 = 런타임 부담. vanilla-extract / CSS Module 검토.
6. **`as` prop 의 type 손실** — `<Button as="a" href="...">` 의 href 타입 추론 약함.

## 13. SSR / Next.js

styled-components v6+ 는 Next.js 14 SSR 지원. CSR-only 인 vite 앱은 신경 안 써도 됨.

## 14. 외부 자료

- [styled-components 공식](https://styled-components.com/)
- [styled-components 의 transient props](https://styled-components.com/docs/api#transient-props)

## 15. 관련

- [[styling]]
- [[tailwind]]
- [[../typescript/component-types]]
