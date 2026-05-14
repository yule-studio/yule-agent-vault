---
title: "Stale Closures — useEffect / setInterval 안 옛 state"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, pitfalls, closure, stale-state, useeffect]
---

# Stale Closures

**[[pitfalls|↑ Pitfalls Hub]]**

> "왜 state 가 0 그대로?" 의 99% 원인. closure 안 옛 변수.

## 1. 가장 흔한 케이스

```tsx
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1);     // ❌ count 는 영원히 0
    }, 1000);
    return () => clearInterval(id);
  }, []);     // deps 비어 있음

  return <p>{count}</p>;
}
```

→ `setInterval` 의 callback 이 마운트 시점의 `count = 0` 을 capture. 1초마다 `setCount(0 + 1) = 1`. count 가 1 이 된 후에도 callback 안 count 는 여전히 0.

## 2. 왜 그렇게 되는가?

JavaScript 의 closure:
```js
let x = 0;
const fn = () => console.log(x);
fn();      // 0
x = 1;
fn();      // 1 — 같은 변수 참조
```

React 는 매 render 마다 **새로운 함수 + 새로운 변수**:
```tsx
function Counter() {
  const [count, setCount] = useState(0);     // 매 render 새 count
  
  // 이 함수도 매 render 새로 만들어짐
  const handler = () => console.log(count);
  
  return <button onClick={handler}>{count}</button>;
}
```

→ click 핸들러는 그 render 의 count 를 capture. 정상.

문제는 `useEffect(() => ..., [])` — effect 가 마운트 시 1 번만 실행 → 그 안 함수가 **마운트 시점의 변수** 를 영원히 capture.

## 3. 해결 1 — 함수형 update

```tsx
useEffect(() => {
  const id = setInterval(() => {
    setCount(c => c + 1);     // ✅ 항상 최신 count
  }, 1000);
  return () => clearInterval(id);
}, []);
```

→ React 가 호출 시점의 state 를 함수에 넘김. 가장 단순.

## 4. 해결 2 — deps 에 추가

```tsx
useEffect(() => {
  const id = setInterval(() => {
    setCount(count + 1);
  }, 1000);
  return () => clearInterval(id);
}, [count]);    // count 바뀔 때마다 effect 재설정
```

→ count 변경마다 interval 재설정 = 비효율. 함수형 update 가 더 나음.

## 5. 해결 3 — useRef 로 최신 값 보관

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  const countRef = useRef(count);

  useEffect(() => {
    countRef.current = count;     // 매 render 갱신
  });

  useEffect(() => {
    const id = setInterval(() => {
      setCount(countRef.current + 1);
    }, 1000);
    return () => clearInterval(id);
  }, []);
}
```

→ ref 는 변경되어도 re-render 안 함. 외부 lib callback 안 stale 해결의 표준.

## 6. event handler 안 stale

```tsx
function SearchBox() {
  const [query, setQuery] = useState('');

  useEffect(() => {
    document.addEventListener('keydown', (e) => {
      if (e.key === 'Enter') {
        console.log(query);    // ❌ 영원히 '' (마운트 시점)
      }
    });
  }, []);
}
```

```tsx
// 해결 — deps 에 query
useEffect(() => {
  const handler = (e) => {
    if (e.key === 'Enter') console.log(query);
  };
  document.addEventListener('keydown', handler);
  return () => document.removeEventListener('keydown', handler);
}, [query]);     // 매 변경 시 reattach
```

→ 깔끔. 또는 ref 패턴.

## 7. setTimeout / Promise 안

```tsx
function Comp() {
  const [user, setUser] = useState(null);

  const fetchUser = () => {
    setUser({ name: 'temp' });
    setTimeout(() => {
      console.log(user);   // ❌ null
    }, 100);
  };
}
```

→ `setUser` 호출 후 user 는 다음 render 의 새 변수. `setTimeout` 의 callback 은 이 render 의 user (null) 만 봄.

```tsx
// 해결 — useEffect 로 다음 render
useEffect(() => {
  if (user) console.log(user);
}, [user]);
```

## 8. async function 안 stale

```tsx
function Comp() {
  const [count, setCount] = useState(0);

  const slowSave = async () => {
    const initial = count;   // capture 시점의 count
    await sleep(2000);
    api.save(initial);        // 옛 값. 사용자가 그 사이 count 바꿔도 안 반영.
  };
}
```

→ 의도면 OK. "현재 값" 원하면 ref + 갱신:
```tsx
const countRef = useRef(count);
useEffect(() => { countRef.current = count; });

const slowSave = async () => {
  await sleep(2000);
  api.save(countRef.current);   // 호출 시점의 최신
};
```

## 9. custom hook 안 stale

```tsx
// useEventListener custom hook
function useEventListener(eventName, handler) {
  useEffect(() => {
    window.addEventListener(eventName, handler);
    return () => window.removeEventListener(eventName, handler);
  }, [eventName]);    // handler 는 deps 에 없음
}

// 사용
function Comp() {
  const [count, setCount] = useState(0);
  useEventListener('click', () => console.log(count));  // count 영원히 0
}
```

→ ref pattern 으로 해결 (handler 의 최신 reference 보관):
```ts
function useEventListener(eventName, handler) {
  const savedHandler = useRef(handler);
  useEffect(() => { savedHandler.current = handler; }, [handler]);
  useEffect(() => {
    const listener = (e) => savedHandler.current(e);
    window.addEventListener(eventName, listener);
    return () => window.removeEventListener(eventName, listener);
  }, [eventName]);
}
```

→ usehooks-ts 등 라이브러리의 표준 패턴.

## 10. axios interceptor 안 stale token

```ts
// ❌
api.interceptors.request.use((config) => {
  const token = useAuth.getState().token;   // 좋음
});

// ❌ 다른 패턴
function setupApi() {
  const token = useAuth.getState().token;    // 한 번만 capture → stale
  api.interceptors.request.use((config) => {
    config.headers.Authorization = `Bearer ${token}`;
    return config;
  });
}
```

→ interceptor 안에서 매번 storage / store 에서 read.

## 11. React 18 의 `useEffectEvent` (experimental)

```tsx
// experimental — React 18.3+
import { experimental_useEffectEvent as useEffectEvent } from 'react';

const onTick = useEffectEvent(() => {
  console.log(count);    // 항상 최신
});

useEffect(() => {
  const id = setInterval(onTick, 1000);
  return () => clearInterval(id);
}, []);
```

→ deps 에 안 들어가지만 최신 closure. React 의 공식 해결책. 정식 출시 대기.

## 12. 진단 — stale closure 의심?

1. console.log 로 callback 안 값 출력.
2. 첫 마운트 시 값과 같다면 stale.
3. eslint `react-hooks/exhaustive-deps` 가 보통 warning.

## 13. 함정 정리

| 상황 | 해결 |
| --- | --- |
| setInterval / setTimeout 안 state | 함수형 update, ref |
| Effect 의 event listener | deps 추가, ref |
| async 안 옛 값 | ref + useEffect 로 sync |
| custom hook 의 handler | ref pattern |
| Interceptor 안 token | 매번 storage read |

## 14. 외부 자료

- [Dan Abramov "A Complete Guide to useEffect"](https://overreacted.io/a-complete-guide-to-useeffect/) — 필독.
- [react.dev — useEffectEvent](https://react.dev/learn/separating-events-from-effects)

## 15. 관련

- [[pitfalls]]
- [[re-render-loops]]
- [[../core-concepts/hooks-essentials]]
