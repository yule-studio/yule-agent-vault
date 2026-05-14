---
title: "Hooks Essentials — useState · useEffect · useRef · useMemo · useCallback"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, hooks, useeffect, useref, beginner]
---

# Hooks Essentials

**[[core-concepts|↑ Core Concepts]]**

> Hook = `use` 로 시작하는 특별 함수. React 의 모든 기능 (state, lifecycle, ref ...) 의 진입점.

## 1. Hook 의 규칙 (절대 깨지 마세요)

1. **Top-level 에서만** — if / for / 함수 안에서 호출 금지.
2. **함수 컴포넌트 또는 다른 custom hook 에서만**.
3. **매 render 호출 순서 동일**.

```tsx
// ❌ 조건 안
if (user) {
  const [x, setX] = useState(0);   // 깨짐
}

// ✅
const [x, setX] = useState(0);
if (!user) return null;
```

→ ESLint `react-hooks/rules-of-hooks` 가 자동 검사.

## 2. useState — 이미 학습 ([[props-and-state]])

```tsx
const [count, setCount] = useState(0);
```

## 3. useEffect — side effect 의 진입점

```tsx
useEffect(() => {
  // side effect (API 호출, subscription, DOM 직접 조작)
  console.log('마운트 또는 deps 변경');

  return () => {
    // cleanup (언마운트 또는 다음 effect 직전)
    console.log('cleanup');
  };
}, [deps]);  // dependency array
```

### Deps 의 3 가지 모드

```tsx
// 1. deps 없음 — 매 render 마다 실행 (거의 안 씀)
useEffect(() => { ... });

// 2. [] — 마운트 시 1 번
useEffect(() => {
  console.log('첫 마운트');
}, []);

// 3. [x, y] — x 또는 y 가 바뀔 때마다
useEffect(() => {
  console.log('x or y changed');
}, [x, y]);
```

### 흔한 사용 예

```tsx
// (1) 마운트 시 API 호출
useEffect(() => {
  fetch('/api/user')
    .then(r => r.json())
    .then(data => setUser(data));
}, []);

// (2) prop 변경 시 재호출
useEffect(() => {
  fetch(`/api/user/${userId}`)
    .then(r => r.json())
    .then(setUser);
}, [userId]);

// (3) subscription + cleanup
useEffect(() => {
  const id = setInterval(() => console.log('tick'), 1000);
  return () => clearInterval(id);    // 언마운트 시 정리
}, []);

// (4) event listener
useEffect(() => {
  const handler = () => console.log('scroll');
  window.addEventListener('scroll', handler);
  return () => window.removeEventListener('scroll', handler);
}, []);
```

### Strict Mode 의 double mount

```tsx
// React 18 Strict Mode 에서 dev 모드는 useEffect 가 2 번 실행
// → cleanup 작성 강제용. 실제 production 은 1 번.
```

## 4. useRef — DOM 또는 mutable 값

### (A) DOM ref

```tsx
function Focus() {
  const inputRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    inputRef.current?.focus();   // 마운트 후 자동 focus
  }, []);

  return <input ref={inputRef} />;
}
```

### (B) mutable 값 보관

```tsx
function Timer() {
  const startTime = useRef(Date.now());     // re-render 영향 받지 않음
  // startTime.current 로 read/write
}
```

→ ref 는 변경해도 **re-render 안 함**. state 와의 차이.

| | useState | useRef |
| --- | --- | --- |
| 값 변경 | setX | `.current = ...` |
| re-render | O | X |
| 비유 | "보여줄 값" | "그냥 보관" |

## 5. useMemo — 값 캐싱

```tsx
function ExpensiveList({ items, filter }) {
  // 비싼 계산 — items 또는 filter 가 안 바뀌면 캐시
  const filtered = useMemo(() => {
    return items.filter(it => it.name.includes(filter));
  }, [items, filter]);

  return <ul>{filtered.map(...)}</ul>;
}
```

→ deps 가 같으면 이전 계산 결과 재사용.

## 6. useCallback — 함수 캐싱

```tsx
function Parent() {
  const [count, setCount] = useState(0);

  // ❌ 매 render 마다 새 함수 → 자식 re-render
  const handleClick = () => setCount(c => c + 1);

  // ✅ 같은 deps 면 같은 함수 참조 유지
  const handleClick = useCallback(() => setCount(c => c + 1), []);

  return <Child onClick={handleClick} />;
}
```

→ `Child` 가 `React.memo` 일 때 prop 참조 동일 → re-render skip.

## 7. useMemo vs useCallback

```tsx
useMemo(() => fn, [deps])        // fn 자체의 return 값 캐시
useCallback(fn, [deps])           // fn 자체를 캐시

useCallback(fn, deps) === useMemo(() => fn, deps)
```

→ 동등. useCallback 은 함수 전용 shortcut.

## 8. useReducer — 복잡 state 정리

```tsx
type State = { count: number };
type Action = { type: 'inc' } | { type: 'dec' } | { type: 'reset' };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'inc': return { count: state.count + 1 };
    case 'dec': return { count: state.count - 1 };
    case 'reset': return { count: 0 };
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0 });
  return (
    <>
      <p>{state.count}</p>
      <button onClick={() => dispatch({ type: 'inc' })}>+</button>
      <button onClick={() => dispatch({ type: 'reset' })}>0</button>
    </>
  );
}
```

→ useState 가 너무 복잡해질 때. Redux 의 reducer 와 유사.

## 9. useContext — 전역 값

```tsx
const ThemeContext = React.createContext('light');

function App() {
  return (
    <ThemeContext.Provider value="dark">
      <Page />
    </ThemeContext.Provider>
  );
}

function Page() {
  const theme = useContext(ThemeContext);    // 'dark'
  return <p>{theme}</p>;
}
```

→ props drilling 해결.
**단 자주 변경되는 값은 비추** (Context 안 모든 소비자 re-render). 외부 store (zustand 등) 권장.

## 10. Custom Hook — 자기만의 hook

```tsx
function useToggle(initial = false) {
  const [value, setValue] = useState(initial);
  const toggle = useCallback(() => setValue(v => !v), []);
  return [value, toggle] as const;
}

// 사용
function Comp() {
  const [isOpen, toggle] = useToggle();
  return <button onClick={toggle}>{isOpen ? 'Open' : 'Closed'}</button>;
}
```

→ `use` 로 시작하는 함수 = custom hook. 안에 다른 hook 호출 OK.

### 실전 — useFetch

```tsx
function useFetch<T>(url: string) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    let cancelled = false;
    setLoading(true);
    fetch(url)
      .then(r => r.json())
      .then(d => !cancelled && setData(d))
      .catch(e => !cancelled && setError(e))
      .finally(() => !cancelled && setLoading(false));
    return () => { cancelled = true; };
  }, [url]);

  return { data, loading, error };
}
```

→ 실제 프로젝트는 [[../server-state/react-query|TanStack Query]] 권장.

## 11. answer-fe / job-answer-fe 의 흔한 hook

`masterway-dev/answer-fe/src/hooks/` 살펴보기:
```bash
ls ~/masterway-dev/answer-fe/src/hooks/
```

전형적 custom hook:
- `useDebounce` — 입력 debounce.
- `useModal` — 모달 상태.
- `useInfiniteScroll` — 무한 스크롤.
- `useClickOutside` — 외부 클릭 감지.

## 12. 함정

1. **Deps 누락** — `useEffect(() => { fetch(`/api/${id}`) }, [])` 가 `id` 변경 시 무효. ESLint `react-hooks/exhaustive-deps` 가 검사.
2. **Object / function 을 deps 에** — 매 render 새 참조 → effect 매번 실행. useMemo / useCallback 으로 안정화.
3. **Stale closure** — [[../pitfalls/stale-closures]]
   ```tsx
   useEffect(() => {
     const id = setInterval(() => setCount(count + 1), 1000);  // count 옛 값
     return () => clearInterval(id);
   }, []);
   // → setCount(c => c + 1) 로 해결
   ```
4. **무한 loop** — `useEffect(() => setX(...))` deps 없으면 무한.
5. **Cleanup 누락** — interval / subscription / abortController 정리 안 하면 메모리 누수.
6. **Conditional hook** — if 안에서 useState 호출 ❌.
7. **useMemo / useCallback 남용** — 단순 값이면 비용이 더 큼. 측정 후 적용.

## 13. 정리 표

| Hook | 언제 |
| --- | --- |
| useState | 단순 state |
| useReducer | 복잡 state 전환 |
| useEffect | 마운트 / 변경 시 side effect |
| useRef | DOM 또는 변경해도 re-render 불요 값 |
| useMemo | 비싼 계산 캐시 |
| useCallback | 함수 참조 캐시 (memo 자식에 전달 시) |
| useContext | 전역 값 (드물게 변경) |
| custom hook | 로직 재사용 |

## 14. 다음 단계

- [[conditional-and-list-rendering]] — 조건 / 리스트 렌더링
- [[../pitfalls/stale-closures]]
- [[../server-state/react-query]]

## 15. 관련

- [[core-concepts]]
- [[../pitfalls/re-render-loops]]
- [[../performance/memoization]]
