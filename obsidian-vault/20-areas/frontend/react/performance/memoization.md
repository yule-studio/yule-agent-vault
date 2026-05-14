---
title: "Memoization — React.memo / useMemo / useCallback"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, performance, memo, usememo, usecallback]
---

# Memoization

**[[performance|↑ Performance Hub]]**

> 같은 입력 → 같은 출력 이면 cache. **3 도구**: `React.memo`, `useMemo`, `useCallback`.

## 1. React 의 기본 — 부모 re-render → 자식도

```tsx
function Parent() {
  const [count, setCount] = useState(0);
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Child />     {/* count 와 무관해도 매번 re-render */}
    </>
  );
}

function Child() {
  console.log('Child render');
  return <p>I am child</p>;
}
```

→ Parent 가 re-render 하면 Child 도 자동 re-render.
**대부분 문제 X**. 하지만 Child 가 무거우면 최적화.

## 2. React.memo — props 같으면 skip

```tsx
const Child = React.memo(function Child() {
  console.log('Child render');
  return <p>...</p>;
});
```

→ props 가 shallow equal 이면 re-render skip.

### props 가 있는 경우

```tsx
const Child = React.memo(function Child({ name }: { name: string }) {
  return <p>{name}</p>;
});

function Parent() {
  const [count, setCount] = useState(0);
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Child name="유철" />     {/* name 안 바뀜 → re-render skip */}
    </>
  );
}
```

### custom 비교

```tsx
const Child = React.memo(
  function Child({ user }) { ... },
  (prev, next) => prev.user.id === next.user.id      // id 만 비교
);
```

## 3. memo 무용지물 — 매번 새 객체 / 함수 prop

```tsx
function Parent() {
  const [count, setCount] = useState(0);
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Child
        items={[1, 2, 3]}                  // 매 render 새 배열
        handler={() => console.log('hi')}  // 매 render 새 함수
      />
    </>
  );
}
```

→ `items` / `handler` 의 참조가 매번 달라짐 → memo 무력화.

### 해결 — useMemo / useCallback

```tsx
function Parent() {
  const [count, setCount] = useState(0);
  const items = useMemo(() => [1, 2, 3], []);
  const handler = useCallback(() => console.log('hi'), []);
  return <Child items={items} handler={handler} />;
}
```

## 4. useMemo — 값 캐시

```tsx
function ExpensiveList({ items, filter }) {
  const filtered = useMemo(
    () => items.filter(it => it.name.includes(filter)),
    [items, filter]
  );
  return <ul>{filtered.map(...)}</ul>;
}
```

→ items 또는 filter 변경 시에만 재계산.

### 언제 useMemo?
- 계산이 **진짜 비쌀 때** (수천 개 filter, 무거운 변환).
- 참조 stable 필요 (memo 자식의 prop).

### 언제 안 써?
- 단순 계산 (덧셈, string concat).
- 매번 변하는 deps.

## 5. useCallback — 함수 캐시

```tsx
function Parent() {
  const [count, setCount] = useState(0);

  const handler = useCallback(() => {
    console.log(count);
  }, [count]);     // count 변할 때만 새 함수

  return <Child onClick={handler} />;
}
```

→ Child 가 React.memo + onClick prop 받을 때 효과.

### useMemo vs useCallback
```ts
useCallback(fn, deps)
===
useMemo(() => fn, deps)
```

→ 동등. useCallback 은 함수용 shortcut.

## 6. 측정 — Profiler 로 확인

```tsx
import { Profiler } from 'react';

<Profiler id="App" onRender={(id, phase, actualDuration) => {
  console.log(`${id} ${phase}: ${actualDuration.toFixed(2)}ms`);
}}>
  <App />
</Profiler>
```

→ 또는 React DevTools Profiler.

## 7. 잘못된 패턴 — 모든 곳 memo

```tsx
// ❌ 의미 없는 적용
const Title = React.memo(({ text }) => <h1>{text}</h1>);

function App() {
  return <Title text="Hello" />;
}
```

→ Title 의 render 자체가 매우 가벼움. memo 비용 > 절약.

**원칙**: 측정 + 무거운 컴포넌트만.

## 8. context + memo

```tsx
const ThemeContext = createContext('light');

const Child = React.memo(() => {
  const theme = useContext(ThemeContext);    // memo 무력 — context 변경 시 항상 re-render
  return <p>{theme}</p>;
});
```

→ context 값 변경되면 모든 소비자 re-render (memo 도 못 막음). context 잘게 분할 또는 zustand 사용.

## 9. List 의 row 최적화

```tsx
const Row = React.memo(({ item, onToggle }) => {
  return <li onClick={() => onToggle(item.id)}>{item.name}</li>;
});

function List({ items }) {
  const onToggle = useCallback((id) => {
    /* ... */
  }, []);

  return items.map(it => <Row key={it.id} item={it} onToggle={onToggle} />);
}
```

→ 한 row 의 state 변경이 다른 row 에 영향 X.

## 10. setState 의 immutable 패턴 + memo

```tsx
const [items, setItems] = useState<Item[]>([]);

// ❌ 같은 array 참조 반환 → memo 자식 변경 못 감지... 가 아닌, React 자체가 변경 못 감지
items.push(newItem);
setItems(items);

// ✅ 새 array
setItems([...items, newItem]);
```

→ React 의 변경 감지 = 참조 비교. memo 와 같은 원칙.

## 11. Reselect / 파생 데이터 캐시

```tsx
// 잦은 selection of derived value
const filtered = useMemo(
  () => users.filter(u => u.active && u.role === role),
  [users, role]
);
```

→ users 안 바뀌고 role 같으면 같은 array 참조. memo 자식에 prop 으로 전달 안전.

## 12. answer-fe 패턴 예시

```tsx
const QuestionItem = React.memo(({ question, onClick }) => {
  return <li onClick={() => onClick(question.id)}>{question.title}</li>;
});

function QuestionList({ questions }) {
  const navigate = useNavigate();
  const onClick = useCallback((id) => navigate(`/questions/${id}`), [navigate]);

  return (
    <ul>
      {questions.map(q => <QuestionItem key={q.id} question={q} onClick={onClick} />)}
    </ul>
  );
}
```

→ list row 의 표준.

## 13. React 19 의 React Compiler (참고)

```
React Compiler 가 자동 memo / useMemo / useCallback 처리.
→ 수동 최적화 필요 감소.
```

→ 정식 출시 후엔 직접 memo 적용 감소 전망. 지금은 수동.

## 14. 함정

1. **deps 누락** — `useMemo(() => fn(x), [])` 에서 x 변경 후 옛 값 cache.
2. **객체 / 함수를 deps 에** — 매 render 새 참조 → 매번 재계산.
3. **모든 곳 memo** — 비용 증가, 절약 X.
4. **memo + context** — context 변경 시 무력.
5. **inline JSX prop** — `<Child el={<X />} />` 매 render 새 JSX. memo 자식이면 무력.
6. **memo 의 props 가 children** — children 도 매 render 새 element.

## 15. 외부 자료

- [react.dev — useMemo](https://react.dev/reference/react/useMemo)
- [react.dev — useCallback](https://react.dev/reference/react/useCallback)
- [TkDodo "Don't Over Use Refs"](https://tkdodo.eu/blog/use-state-vs-use-ref) (관련 패턴)
- [React Compiler](https://react.dev/learn/react-compiler)

## 16. 관련

- [[performance]]
- [[../core-concepts/hooks-essentials]]
- [[../pitfalls/re-render-loops]]
