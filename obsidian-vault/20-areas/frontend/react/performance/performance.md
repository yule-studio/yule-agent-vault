---
title: "Performance — 성능 최적화 길잡이"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, performance, optimization, hub]
---

# Performance

**[[../react|↑ React Hub]]**

> **"느려진 뒤" 최적화**. 우선은 정확한 코드 → 측정 → 핫스팟 최적화.

## 1. 성능 문제의 4 가지 카테고리

| | 영역 | 해결 도구 |
| --- | --- | --- |
| **번들 크기** | 첫 진입 / 로딩 | code-splitting, tree-shaking, dynamic import |
| **렌더링** | 불필요 re-render | React.memo, useMemo, useCallback |
| **큰 list** | 수천 row | 가상화 (react-window) |
| **이미지 / asset** | 큰 파일 | lazy load, WebP, CDN |

## 2. 측정 도구

- **React DevTools — Profiler 탭** — 어느 컴포넌트가 몇 ms 걸리는지.
- **Chrome DevTools — Performance** — 렌더링 + Layout + Script.
- **Lighthouse** — LCP / TBT / CLS 등 web vitals.
- **`why-did-you-render`** — 불필요 re-render 의 원인.
- **`vite-bundle-visualizer`** — 번들 크기 분석.

→ **측정 없이 최적화 ❌**.

## 3. 학습 우선순위

1. **[[memoization]]** — React.memo / useMemo / useCallback.
2. **[[code-splitting]]** — lazy + Suspense.
3. **[[virtual-list]]** — react-window.
4. **이미지 lazy load** — `loading="lazy"` 또는 IntersectionObserver.
5. **Web Vitals**.

## 4. 너무 빠른 최적화의 함정

```tsx
// ❌ 매 컴포넌트마다 React.memo + useMemo + useCallback
const Title = React.memo(({ text }) => {
  const upper = useMemo(() => text.toUpperCase(), [text]);
  return <h1>{upper}</h1>;
});
```

→ memo / useMemo / useCallback 자체에 비용 (compare, dep check, GC).
**측정 결과 hot spot 만 최적화**.

## 5. Bundle 크기 — 첫 로딩

### 측정
```bash
yarn add -D vite-bundle-visualizer
npx vite-bundle-visualizer
```

→ 어느 lib 가 가장 큰지.

### 흔한 범인
- MUI / antd 전체 import.
- lodash 전체 import (`import _ from 'lodash'` ❌, named import ✅).
- moment.js — date-fns / dayjs 가 가벼움.
- 사용 안 하는 SDK / lib.

### 해결
- named import + tree-shaking.
- 큰 페이지 lazy.
- alternative library.

자세히 [[code-splitting]].

## 6. Render 최적화

### 원인 발견
- React DevTools Profiler 로 무엇이 re-render.
- 부모 re-render → 자식도 자동 re-render (props 변경 안 됐어도).

### 해결
- `React.memo` — props 같으면 자식 re-render skip.
- `useMemo` — 비싼 계산 cache.
- `useCallback` — 함수 참조 stable (memo 자식의 prop).
- state 잘 분리 — 위에서 변경되면 위만 re-render.

자세히 [[memoization]].

## 7. 큰 list (가상화)

```tsx
// 1000 개 row
<ul>
  {items.map(item => <Row key={item.id} item={item} />)}
</ul>
```

→ 모든 1000 개 DOM 생성 = 느림. **react-window** 가 보이는 row 만:

```tsx
import { FixedSizeList } from 'react-window';

<FixedSizeList height={400} itemCount={items.length} itemSize={48} width="100%">
  {({ index, style }) => (
    <div style={style}>{items[index].name}</div>
  )}
</FixedSizeList>
```

→ 100 ~ 200 개 정도부터 도입 고려. 자세히 [[virtual-list]].

## 8. 이미지 — lazy + 포맷

```tsx
<img src="..." loading="lazy" decoding="async" />
```

- 브라우저 native lazy load.
- WebP / AVIF 가 JPG / PNG 보다 작음.
- 이미지 size 명시 (`width`, `height`) → CLS 방지.

```tsx
<img
  src="image.webp"
  width={400}
  height={300}
  loading="lazy"
  decoding="async"
  alt="..."
/>
```

## 9. font

```css
/* 기본 — FOIT (안 보임) */
@font-face {
  font-family: 'Pretendard';
  src: url('...') format('woff2');
  font-display: swap;       /* 폰트 로딩 전 시스템 폰트로 표시 */
}
```

→ `font-display: swap` 이 사용자 경험 좋음.

## 10. memo 와 함께 — 안티 패턴 주의

```tsx
const Parent = () => {
  const handler = () => console.log('hi');   // 매 render 새 함수
  return <Child onClick={handler} />;
};

const Child = React.memo(({ onClick }) => {
  // onClick 매 render 새 참조 → memo 무용
  return <button onClick={onClick}>...</button>;
});
```

```tsx
// 해결
const Parent = () => {
  const handler = useCallback(() => console.log('hi'), []);
  return <Child onClick={handler} />;
};
```

## 11. 상태의 위치와 성능

```tsx
// ❌ 전체 페이지 re-render
const Page = () => {
  const [filter, setFilter] = useState('');
  return (
    <>
      <Header />
      <Filter value={filter} onChange={setFilter} />
      <BigList filter={filter} />
    </>
  );
};

// ✅ state 를 BigList 가까이로
const Page = () => {
  return (
    <>
      <Header />
      <FilterableBigList />        {/* 그 안에서 filter state 보유 */}
    </>
  );
};
```

→ state 변경 시 변경 영향 받는 컴포넌트만 re-render. **"state 는 사용처에 가깝게"**.

## 12. React 18 — Concurrent / Transitions

```tsx
import { useTransition } from 'react';

const [isPending, startTransition] = useTransition();

const onChange = (e) => {
  setInput(e.target.value);   // 즉시 UI
  startTransition(() => {
    setQuery(e.target.value); // 무거운 작업 (필터링 등) 우선순위 낮춤
  });
};
```

→ 검색 input 의 즉시 반응 + 결과 list 의 비동기 갱신.

## 13. React Server Components / Next.js 영역

- SSR / SSG: 첫 페이지 로딩 빠르게.
- React Server Component: 클라이언트 번들 더 작게.

→ Vite 의 CSR-only 인 `masterway-dev/-fe` 는 직접 적용 X. SEO 필요 시 Next.js 검토.

## 14. masterway-dev 의 성능 패턴

`answer-fe`:
- react-window 사용 (큰 list).
- styled-components 가 런타임 비용 (정도는 큰 issue X).
- lazy 적용된 페이지 (라우터별 chunk).

```bash
grep -rn "lazy\|React.memo\|useMemo\|useCallback" ~/masterway-dev/answer-fe/src --include="*.tsx" -l | head -10
```

## 15. 함정

1. **측정 없이 최적화** — 비용만 늘림.
2. **memo + 함수 prop** — useCallback 누락 시 무효.
3. **inline object / array** — `<Comp items={[1,2]} />` 매 render 새 array. useMemo.
4. **state 너무 위로** — 모든 자식 re-render.
5. **virtual list 의 동적 height** — VariableSizeList + 측정 필요.
6. **bundle 크기 무시** — 첫 진입 5MB → 모바일에서 1 분 로딩.

## 16. 외부 자료

- [React DevTools Profiler](https://react.dev/learn/react-developer-tools)
- [web.dev — Web Vitals](https://web.dev/vitals/)
- [TkDodo "Things to Know About useState"](https://tkdodo.eu/blog/things-to-know-about-use-state)

## 17. 관련

- [[memoization]]
- [[code-splitting]]
- [[virtual-list]]
- [[../pitfalls/re-render-loops]]
