---
title: "React 함정 모음 — 무한 re-render, stale closure, key 등"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, pitfalls, gotchas, hub]
---

# React 함정 모음

**[[../react|↑ React Hub]]**

> 실무에서 반복적으로 부딪치는 함정. 알고 있으면 디버깅 시간 1/10.

## 1. 카테고리

- **re-render 관련** — 무한 loop, 불필요 re-render. [[re-render-loops]]
- **closure 관련** — stale state, useEffect 안 옛 값. [[stale-closures]]
- **list / key 관련** — key=index 의 state 엉킴. ([[../core-concepts/conditional-and-list-rendering]])
- **타입 / 상태 관련** — undefined / null / falsy 처리.
- **TS / JSX 관련** — 화살표 함수 return / Fragment key.

## 2. 자주 만나는 함정 — TOP 10

### (1) `useEffect` 무한 loop

```tsx
useEffect(() => {
  setCount(count + 1);   // count 변경 → effect 재실행 → 또 +1 → ...
}, [count]);
```

→ effect 안 setState 가 deps 변경하면 무한. [[re-render-loops]].

### (2) Stale closure

```tsx
useEffect(() => {
  const id = setInterval(() => {
    setCount(count + 1);   // count 는 마운트 시 0. 영원히 0+1=1.
  }, 1000);
  return () => clearInterval(id);
}, []);
```

→ 함수형 update (`setCount(c => c + 1)`) 또는 deps 추가. [[stale-closures]].

### (3) `key={index}` 의 state 엉킴

```tsx
{users.map((u, i) => (
  <div key={i}>
    <input defaultValue={u.name} />
  </div>
))}
// user 한 명 삭제 시 key 가 그대로라 React 가 input 의 state 를 엉뚱한 user 에 연결
```

→ 안정적 unique id 사용. ([[../core-concepts/conditional-and-list-rendering]])

### (4) `&&` 의 `0` 표시

```tsx
{items.length && <List items={items} />}
// items.length === 0 일 때 "0" 이 화면에.
```

→ `items.length > 0 && <List />` 또는 `Boolean(items.length)`.

### (5) `||` 의 `0` / `''` 처리

```tsx
const x = value || 'default';
// value === 0 또는 '' 이면 'default' 됨.
```

→ `value ?? 'default'` (nullish coalescing).

### (6) `useState` 직후 옛 값

```tsx
setCount(count + 1);
console.log(count);   // 아직 옛 값
```

→ setState 는 비동기. 이후 render 의 새 값 사용.

### (7) State 직접 mutate

```tsx
items.push(x);
setItems(items);    // 같은 참조 → re-render 안 됨
```

→ `setItems([...items, x])` (immutable).

### (8) Object / Array prop 매번 새로

```tsx
<Child config={{ a: 1, b: 2 }} />
// 매 render 새 객체 → memo 무력
```

→ `useMemo` 또는 컴포넌트 밖 상수.

### (9) Inline 함수 + memo 자식

```tsx
<Child onClick={() => setX(...)} />
// 매 render 새 함수 → memo 자식 무력
```

→ `useCallback`.

### (10) Conditional hook

```tsx
if (user) {
  const [x] = useState(0);   // ❌
}
```

→ Hook 규칙 위반. top-level 만.

## 3. 학습 우선순위

1. **[[re-render-loops]]** — 무한 loop / 잘못된 deps.
2. **[[stale-closures]]** — closure 의 옛 값.
3. **list / key** — [[../core-concepts/conditional-and-list-rendering]].
4. **memo 함정** — [[../performance/memoization]].

## 4. 디버깅 도구

### React DevTools
- 컴포넌트 tree 확인.
- props / state 실시간 확인.
- Profiler 로 render 추적.

### `why-did-you-render`
```bash
yarn add @welldone-software/why-did-you-render
```
```ts
// wdyr.ts
import React from 'react';
if (process.env.NODE_ENV === 'development') {
  const whyDidYouRender = require('@welldone-software/why-did-you-render');
  whyDidYouRender(React, { trackAllPureComponents: true });
}
```

→ 불필요 re-render 원인 console 에 표시.

### ESLint react-hooks 플러그인
- `react-hooks/rules-of-hooks` — conditional hook 잡기.
- `react-hooks/exhaustive-deps` — useEffect deps 잡기.

```js
// eslint.config.js
import reactHooks from 'eslint-plugin-react-hooks';

export default [
  {
    plugins: { 'react-hooks': reactHooks },
    rules: reactHooks.configs.recommended.rules,
  },
];
```

→ 필수.

## 5. 메모리 누수 (cleanup 누락)

```tsx
// 흔한 누수
useEffect(() => {
  const id = setInterval(() => fetch('/api'), 1000);
  // ❌ cleanup 없음
}, []);

useEffect(() => {
  window.addEventListener('scroll', handler);
  // ❌ cleanup 없음
}, []);
```

→ unmount 시 cleanup 필수:
```tsx
return () => {
  clearInterval(id);
  window.removeEventListener('scroll', handler);
};
```

## 6. 컴포넌트 안 컴포넌트 정의

```tsx
function Parent() {
  function Child() { ... }    // 매 render 새 컴포넌트
  return <Child />;
}
```

→ Child 의 state 매 render 손실 + memo 무력. **컴포넌트 밖** 정의.

## 7. setState 의 batch

```tsx
const handleClick = () => {
  setA(1);
  setB(2);
  setC(3);
};
// React 18+ : 자동 batch (1 회 re-render)
// React 17: event 안만 batch, setTimeout 안은 N 회
```

→ React 18 부터 거의 신경 안 써도 됨.

## 8. Strict Mode 의 double mount

```tsx
<React.StrictMode>
  <App />
</React.StrictMode>
```

- Dev 모드에서 useEffect / state 가 2 번 실행.
- cleanup 작성 안 되어 있으면 문제 노출.
- production 에선 1 회.

→ "내 코드만 잘 작성하면" 문제 안 됨.

## 9. Async / setState 의 unmount race

```tsx
useEffect(() => {
  fetchUser().then(setUser);   // unmount 후 setState ❌
}, []);

// 해결
useEffect(() => {
  let cancelled = false;
  fetchUser().then(u => !cancelled && setUser(u));
  return () => { cancelled = true; };
}, []);
```

→ react-query 사용 시 자동 cancel.

## 10. props 변경 시 state 안 update

```tsx
function Comp({ user }) {
  const [name, setName] = useState(user.name);    // ❌ user 바뀌어도 name 유지
  return <input value={name} ... />;
}

// 해결 1: key 로 강제 remount
<Comp key={user.id} user={user} />

// 해결 2: useEffect
useEffect(() => { setName(user.name) }, [user.id]);
```

→ Derived state 의 함정. 가능하면 prop 직접 사용.

## 11. 외부 자료

- [TkDodo's blog](https://tkdodo.eu/blog/) — React 함정 보물 창고.
- [Kent C. Dodds blog](https://kentcdodds.com/blog) — 패턴 + 함정.
- [react.dev — Common Pitfalls](https://react.dev/learn) (각 hook page).

## 12. 관련

- [[re-render-loops]]
- [[stale-closures]]
- [[../performance/memoization]]
- [[../core-concepts/hooks-essentials]]
