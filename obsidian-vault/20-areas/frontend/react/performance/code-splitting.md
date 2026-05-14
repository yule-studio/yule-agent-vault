---
title: "Code Splitting — lazy + Suspense"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, performance, code-splitting, lazy, dynamic-import]
---

# Code Splitting

**[[performance|↑ Performance Hub]]**

> 첫 번들에 모든 코드 X. 필요할 때 chunk 로 분리 로딩.

## 1. 왜 필요한가?

- 첫 진입 시 JS 5MB → 모바일에서 느림.
- 사용자는 처음에 1 페이지만 본다 → 다른 페이지 코드 미리 로딩 불요.

```
모든 페이지 1 chunk (5MB) → 첫 진입 느림
        ↓ code-splitting
첫 페이지 (300KB) + Lazy chunk (각 100~500KB)
→ 첫 진입 빠름, 다른 페이지 진입 시 chunk 로딩
```

## 2. React.lazy + Suspense

```tsx
import { lazy, Suspense } from 'react';

const Dashboard = lazy(() => import('./pages/Dashboard'));
const Profile = lazy(() => import('./pages/Profile'));

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<Spinner />}>
        <Routes>
          <Route path="/" element={<Home />} />          {/* 일반 */}
          <Route path="/dashboard" element={<Dashboard />} />   {/* lazy */}
          <Route path="/profile" element={<Profile />} />        {/* lazy */}
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}
```

→ `Dashboard` 의 코드는 사용자가 `/dashboard` 진입 시 다운로드.

## 3. lazy 의 default export 요구

```tsx
// ✅
const Dashboard = lazy(() => import('./pages/Dashboard'));
// 그 파일: export default Dashboard;

// named export 도 가능 — 약간 trick
const Dashboard = lazy(() =>
  import('./pages/Dashboard').then(m => ({ default: m.Dashboard }))
);
```

## 4. Suspense fallback 배치

```tsx
// 페이지 전체 fallback
<Suspense fallback={<PageSpinner />}>
  <Routes>...</Routes>
</Suspense>

// 또는 페이지마다
<Route path="/x" element={
  <Suspense fallback={<...>}>
    <X />
  </Suspense>
} />
```

→ 보통 root Suspense 1 개. 페이지 단위가 깔끔.

## 5. dynamic import 일반 — 컴포넌트 외 코드

```tsx
const handleClick = async () => {
  // 큰 라이브러리를 필요할 때만 로딩
  const { default: html2canvas } = await import('html2canvas');
  const canvas = await html2canvas(ref.current);
  // ...
};

<button onClick={handleClick}>이미지로 저장</button>
```

→ html2canvas (300KB+) 가 첫 번들에 안 들어감. 클릭 시 로딩.

## 6. preload — 진입 전 미리 받기

```tsx
const Dashboard = lazy(() => import('./pages/Dashboard'));

// 마우스 hover 시 미리 받기
<Link
  to="/dashboard"
  onMouseEnter={() => import('./pages/Dashboard')}
>
  Dashboard
</Link>
```

→ link hover 시 미리 다운로드 → 실제 클릭 시 즉시.

```tsx
// Vite 의 dynamic prefetch
// vite.config.ts 의 build.rollupOptions.experimentalMinChunkSize 등으로 조정
```

## 7. chunk 이름 지정

```tsx
const Dashboard = lazy(() =>
  import(/* webpackChunkName: "dashboard" */ './pages/Dashboard')
);
```

→ Webpack 시절 magic comment. Vite 는 약간 다름 (`chunks` 옵션).

## 8. lazy 의 에러 처리

```tsx
import { ErrorBoundary } from 'react-error-boundary';

<ErrorBoundary fallback={<p>로딩 실패</p>}>
  <Suspense fallback={<Spinner />}>
    <Lazy />
  </Suspense>
</ErrorBoundary>
```

→ chunk 다운로드 실패 (네트워크) 시.

### 자동 재시도

```tsx
const lazyWithRetry = (importer, retries = 3) =>
  lazy(() =>
    importer().catch(async (e) => {
      if (retries > 0) {
        await new Promise(r => setTimeout(r, 500));
        return lazyWithRetry(importer, retries - 1);
      }
      throw e;
    })
  );

const Dashboard = lazyWithRetry(() => import('./pages/Dashboard'));
```

## 9. Library 별 chunk

```ts
// vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'react-vendor': ['react', 'react-dom', 'react-router-dom'],
          'mui': ['@mui/material', '@mui/icons-material'],
          'utils': ['lodash', 'date-fns'],
        },
      },
    },
  },
});
```

→ vendor 별 분리. 라이브러리 자주 변경 X → 캐시 효과.

## 10. tree-shaking 의 보장

```ts
// ❌ 전체 import
import _ from 'lodash';
const x = _.debounce(...);

// ✅ named (tree-shake)
import { debounce } from 'lodash';

// ✅ submodule import (작은 lib)
import debounce from 'lodash/debounce';
```

→ Vite + ESM 은 named import 자동 tree-shake. CJS 는 안 됨.

## 11. 번들 분석

```bash
yarn add -D vite-bundle-visualizer
npx vite-bundle-visualizer
```

→ 어느 lib / 어느 페이지가 큰지 시각화.

## 12. answer-fe / 일반 패턴

```tsx
// router.tsx
import { lazy, Suspense } from 'react';

const HomePage = lazy(() => import('./pages/HomePage'));
const QuestionListPage = lazy(() => import('./pages/QuestionListPage'));
const QuestionDetailPage = lazy(() => import('./pages/QuestionDetailPage'));
const LoginPage = lazy(() => import('./pages/LoginPage'));

const router = (
  <BrowserRouter>
    <Suspense fallback={<PageSpinner />}>
      <Routes>
        <Route path="/" element={<HomePage />} />
        <Route path="/questions" element={<QuestionListPage />} />
        <Route path="/questions/:id" element={<QuestionDetailPage />} />
        <Route path="/login" element={<LoginPage />} />
      </Routes>
    </Suspense>
  </BrowserRouter>
);
```

## 13. 함정

1. **default export 요구** — named 면 `then(m => ({ default: m.X }))`.
2. **Suspense 없이 lazy** — throw promise. 부모 어딘가 Suspense 필요.
3. **chunk 의 캐시 만료** — deploy 후 옛 페이지에서 새 chunk path 못 찾음. `lazyWithRetry` + 페이지 reload fallback.
4. **너무 많은 chunk** — 1 페이지 1 chunk + waterfall 로딩 → 오히려 느림. 도메인 단위로.
5. **inline lazy** — `const X = lazy(...)` 를 컴포넌트 안 정의 X. module 레벨.
6. **모든 페이지 lazy** — 자주 가는 페이지 (홈) 는 inline 권장.

## 14. 외부 자료

- [react.dev — lazy](https://react.dev/reference/react/lazy)
- [Vite manualChunks](https://rollupjs.org/configuration-options/#output-manualchunks)
- [web.dev Code splitting](https://web.dev/reduce-javascript-payloads-with-code-splitting/)

## 15. 관련

- [[performance]]
- [[memoization]]
- [[../routing/react-router-v6]]
