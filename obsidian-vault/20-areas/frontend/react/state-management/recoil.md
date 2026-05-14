---
title: "recoil — Atom + Selector"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, state-management, recoil, atom, selector]
---

# recoil

**[[state-management|↑ State Management Hub]]**

> Facebook 의 atomic state. **유지보수 중단** (last release 2023). `masterway-dev/*-fe` 기존 코드 이해용. 신규 코드는 [[jotai]] 권장.

## 1. 설치

```bash
yarn add recoil
```

## 2. Setup — RecoilRoot 필수

```tsx
import { RecoilRoot } from 'recoil';

function App() {
  return (
    <RecoilRoot>
      <BrowserRouter>...</BrowserRouter>
    </RecoilRoot>
  );
}
```

## 3. atom

```tsx
import { atom, useRecoilState } from 'recoil';

const countState = atom({
  key: 'count',           // 전역 unique key 필수
  default: 0,
});

function Counter() {
  const [count, setCount] = useRecoilState(countState);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

## 4. useRecoilValue / useSetRecoilState — 분리 hook

```tsx
import { useRecoilValue, useSetRecoilState } from 'recoil';

const count = useRecoilValue(countState);          // 읽기만
const setCount = useSetRecoilState(countState);    // 쓰기만 (re-render X)
```

## 5. selector — 파생 상태

```tsx
import { selector, useRecoilValue } from 'recoil';

const doubleCountState = selector({
  key: 'doubleCount',
  get: ({ get }) => get(countState) * 2,
});

function Display() {
  const double = useRecoilValue(doubleCountState);
  return <p>{double}</p>;
}
```

### Writable selector

```tsx
const doubleCountState = selector({
  key: 'doubleCount',
  get: ({ get }) => get(countState) * 2,
  set: ({ set }, newValue) => set(countState, newValue / 2),
});
```

## 6. 비동기 selector

```tsx
const userState = selector({
  key: 'user',
  get: async ({ get }) => {
    const id = get(userIdState);
    const r = await fetch(`/api/users/${id}`);
    return r.json();
  },
});

// 사용 — Suspense 와 결합
<Suspense fallback={<Loading />}>
  <User />
</Suspense>

function User() {
  const user = useRecoilValue(userState);
  return <p>{user.name}</p>;
}
```

→ react-query 가 보통 더 적합 (캐싱 / refetch).

## 7. atomFamily / selectorFamily — 동적 atom

```tsx
import { atomFamily } from 'recoil';

const todoState = atomFamily<Todo, number>({
  key: 'todo',
  default: (id) => ({ id, text: '', done: false }),
});

function Todo({ id }: { id: number }) {
  const [todo, setTodo] = useRecoilState(todoState(id));
  return <input value={todo.text} onChange={...} />;
}
```

## 8. localStorage 영속 — effects

```tsx
import { atom, AtomEffect } from 'recoil';

const localStorageEffect = (key: string): AtomEffect<unknown> => ({ setSelf, onSet }) => {
  const saved = localStorage.getItem(key);
  if (saved !== null) setSelf(JSON.parse(saved));
  onSet((newValue, _, isReset) => {
    isReset
      ? localStorage.removeItem(key)
      : localStorage.setItem(key, JSON.stringify(newValue));
  });
};

const themeState = atom({
  key: 'theme',
  default: 'light',
  effects: [localStorageEffect('theme')],
});
```

## 9. answer-fe / job-answer-fe 패턴

```bash
ls ~/masterway-dev/answer-fe/src/store/
# 또는
grep -rn "atom({" ~/masterway-dev/answer-fe/src --include="*.ts" | head -10
```

전형:
```tsx
// store/userState.ts
export const userState = atom<User | null>({
  key: 'userState',
  default: null,
});

export const tokenState = atom<string | null>({
  key: 'tokenState',
  default: localStorage.getItem('token'),
  effects: [localStorageEffect('token')],
});

export const isLoggedInState = selector({
  key: 'isLoggedIn',
  get: ({ get }) => get(tokenState) !== null,
});
```

## 10. useResetRecoilState — 초기화

```tsx
import { useResetRecoilState } from 'recoil';

const reset = useResetRecoilState(countState);
reset();   // default 로
```

## 11. 함정

1. **key 충돌** — 두 atom 이 같은 key 면 런타임 경고. 보통 file 이름 prefix.
2. **`<RecoilRoot>` 누락** — "must be used within a RecoilRoot" 에러.
3. **selector 의 무한 loop** — selector A 가 B 참조 + B 가 A 참조.
4. **유지보수 중단** — React 18+ 의 Concurrent / Suspense 와 잘 안 맞는 부분 있음. 점진적 jotai 이전 검토.
5. **devtools** — `recoil-devtools` 별도 설치.
6. **bundle size** — 8KB. jotai (2.5KB), zustand (1.5KB) 비해 큼.

## 12. recoil → jotai 마이그레이션

```tsx
// recoil
const countState = atom({ key: 'count', default: 0 });
const doubleState = selector({ key: 'double', get: ({ get }) => get(countState) * 2 });

// jotai
const countAtom = atom(0);
const doubleAtom = atom((get) => get(countAtom) * 2);
```

→ API 거의 동일. key 제거가 가장 큰 차이.

전체 코드 한 번에 못 바꾸면:
- 새 코드는 jotai 로.
- 기존 atom 은 점진 이전 또는 그대로.

## 13. 외부 자료

- [recoil 공식 (보존용)](https://recoiljs.org/)
- [recoil EOL discussion](https://github.com/facebookexperimental/Recoil/issues/2237)

## 14. 관련

- [[state-management]]
- [[jotai]] — 권장 후속
- [[zustand]]
