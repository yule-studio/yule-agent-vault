---
title: "Re-render Loops — 무한 re-render 원인과 해결"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, pitfalls, re-render, infinite-loop, useeffect]
---

# Re-render Loops

**[[pitfalls|↑ Pitfalls Hub]]**

> "Maximum update depth exceeded" 또는 브라우저 멈춤. 패턴 6 가지.

## 1. 무한 loop 의 원리

```
state 변경 → 컴포넌트 re-render → effect 재실행 → state 또 변경 → ...
```

→ effect / render 안에서 그 effect / render 의 input 을 바꾸면 loop.

## 2. 패턴 1 — useEffect deps 누락 + state 변경

```tsx
// ❌
useEffect(() => {
  setCount(count + 1);
});   // deps 없음 = 매 render 마다 → 무한
```

```tsx
// ✅
useEffect(() => {
  setCount(count + 1);
}, []);   // 마운트 시만
```

## 3. 패턴 2 — deps 안 state 를 setState

```tsx
// ❌
useEffect(() => {
  setCount(count + 1);   // count 변경 → effect 재실행 → 또 +1 → ...
}, [count]);
```

```tsx
// ✅ 조건부
useEffect(() => {
  if (count < 10) setCount(count + 1);
}, [count]);

// ✅ 또는 state 가 아닌 deps 만 reactive
useEffect(() => {
  setCount(c => c + 1);   // 한 번만 fire 되도록 deps 조정
}, [someOtherDep]);
```

## 4. 패턴 3 — 객체 / 배열 prop 을 deps 에

```tsx
// ❌
function Comp({ config }) {
  useEffect(() => {
    fetch('/api', { body: JSON.stringify(config) });
  }, [config]);    // 부모가 매 render 새 객체 → 매번 fetch → 무한 가능
}
```

```tsx
// ✅ 1. 부모에서 useMemo
const config = useMemo(() => ({ a, b }), [a, b]);

// ✅ 2. 자식에서 primitive 만 deps
useEffect(() => {
  fetch('/api', { body: JSON.stringify({ a, b }) });
}, [a, b]);
```

## 5. 패턴 4 — render 안 setState

```tsx
function Comp({ data }) {
  // ❌ render 안 직접 setState
  if (data.length > 0) {
    setFiltered(data.filter(...));   // 무한
  }
  return <ul>...</ul>;
}
```

```tsx
// ✅ useEffect
useEffect(() => {
  setFiltered(data.filter(...));
}, [data]);

// ✅ 더 좋은 — derive (state 안 만들기)
const filtered = data.filter(...);   // 매 render 계산 (또는 useMemo)
```

## 6. 패턴 5 — useState 의 초기값 함수 호출

```tsx
// ❌ 사실 무한은 아니지만 매 render 비싼 호출
const [x, setX] = useState(expensiveCompute());

// ✅ lazy init
const [x, setX] = useState(() => expensiveCompute());
```

→ 이건 loop 는 아니지만 비효율. 자주 보임.

## 7. 패턴 6 — 부모-자식 effect 핑퐁

```tsx
// ❌
function Parent() {
  const [val, setVal] = useState(0);
  return <Child val={val} onChange={setVal} />;
}

function Child({ val, onChange }) {
  useEffect(() => {
    onChange(val + 1);   // 매 변경 시 부모 setState → 다시 prop 으로 → 또 effect → 무한
  }, [val]);
}
```

```tsx
// ✅ 명확한 trigger (event) 로만 호출
function Child({ val, onChange }) {
  return <button onClick={() => onChange(val + 1)}>+</button>;
}
```

## 8. 흔한 trigger — derived state

```tsx
function UserCard({ user }) {
  const [name, setName] = useState(user.name);

  // ❌ user 변경 시 name 자동 update
  useEffect(() => {
    setName(user.name);
  }, [user]);

  // user 가 매번 새 객체 (부모가 useMemo 안 함) → 매 render setName → 무한 가능
}
```

→ 해결: derived state 없애기.
```tsx
// 그냥 prop 사용
return <input defaultValue={user.name} />;

// 또는 key 로 강제 remount
<UserCard key={user.id} user={user} />
```

## 9. 디버깅 — 어디서 무한인가?

### React DevTools Profiler
1. Profile 켜고 record.
2. 무한 render 컴포넌트 highlight.
3. "Why did this render" 보기.

### console.log 추가

```tsx
function Comp() {
  console.log('Comp render');     // 1초에 수백 번 찍히면 잘못된 거.
  useEffect(() => {
    console.log('Comp effect');
  });
}
```

### why-did-you-render

```bash
yarn add @welldone-software/why-did-you-render
```

```ts
Comp.whyDidYouRender = true;
```

→ console 에 변경된 prop / state 표시.

### React 의 "Maximum update depth exceeded"

```
Maximum update depth exceeded. This can happen when a component repeatedly
calls setState inside componentWillUpdate or componentDidUpdate.
```

→ React 가 자동 50+ 회 detect 후 throw. 메시지 + stack trace 보고 fix.

## 10. 패턴 정리 — 안 만들기

| 안티 패턴 | 해결 |
| --- | --- |
| useEffect deps 누락 | `[]` 또는 정확한 deps |
| state 가 deps + state 변경 | 함수형 update, 조건부 |
| 객체 prop 을 deps | useMemo, primitive 만 |
| render 안 setState | useEffect 또는 derive |
| Effect 핑퐁 | event 로만 |
| Derived state | prop 사용 또는 key remount |

## 11. ESLint 가 잡아줌

```bash
yarn add -D eslint-plugin-react-hooks
```

```js
// eslint.config.js
{ rules: { 'react-hooks/exhaustive-deps': 'warn' } }
```

→ deps 누락 / mismatch 경고. 거의 모든 React 프로젝트 권장.

## 12. setState 의 시점

```tsx
// React batch
setA(1);
setB(2);
// → 1 회 re-render (React 18 +)
```

→ React 18 부터는 setTimeout / Promise 안에서도 자동 batch.

## 13. 함수형 update — 안전한 setState

```tsx
// ❌ stale state 가능
setCount(count + 1);

// ✅ 이전 값 보장
setCount(c => c + 1);
```

→ async / 여러 setState 동시 호출 시 안전.

## 14. 외부 자료

- [react.dev — You might not need useEffect](https://react.dev/learn/you-might-not-need-an-effect)
- [TkDodo "When useEffect is wrong"](https://tkdodo.eu/blog/myths-about-useeffect)

## 15. 관련

- [[pitfalls]]
- [[stale-closures]]
- [[../core-concepts/hooks-essentials]]
- [[../performance/memoization]]
