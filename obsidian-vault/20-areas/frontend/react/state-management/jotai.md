---
title: "jotai — Atomic State"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, state-management, jotai, atomic]
---

# jotai

**[[state-management|↑ State Management Hub]]**

> Atom = useState 의 전역 버전. `masterway-dev/answer-fe` 가 사용.

## 1. 설치

```bash
yarn add jotai
```

→ Provider 필요 없음 (optional). zustand 처럼 import 하면 끝.

## 2. 가장 작은 예

```tsx
import { atom, useAtom } from 'jotai';

const countAtom = atom(0);   // 어디서나 import 해 같은 atom

function Counter() {
  const [count, setCount] = useAtom(countAtom);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

function Display() {
  const [count] = useAtom(countAtom);
  return <p>{count}</p>;
}
```

→ `Counter` 클릭하면 `Display` 도 자동 업데이트.

## 3. atom 의 종류

### (A) Primitive atom

```tsx
const nameAtom = atom('');
const ageAtom = atom(0);
const userAtom = atom<User | null>(null);
const itemsAtom = atom<string[]>([]);
```

### (B) Derived atom (read-only)

```tsx
const firstNameAtom = atom('철');
const lastNameAtom = atom('유');

const fullNameAtom = atom((get) => `${get(lastNameAtom)}${get(firstNameAtom)}`);

function Name() {
  const [fullName] = useAtom(fullNameAtom);   // 자동 계산
  return <p>{fullName}</p>;
}
```

→ 의존 atom 이 바뀌면 자동 recompute.

### (C) Read-write derived

```tsx
const countAtom = atom(0);

const doubleAtom = atom(
  (get) => get(countAtom) * 2,                    // read
  (get, set, newValue: number) => set(countAtom, newValue / 2)  // write
);
```

### (D) Async atom

```tsx
const userAtom = atom(async (get) => {
  const id = get(userIdAtom);
  const r = await fetch(`/api/users/${id}`);
  return r.json();
});

function User() {
  const [user] = useAtom(userAtom);   // Suspense 와 결합
  return <p>{user.name}</p>;
}
```

→ Suspense fallback 자동. 단 [[../server-state/react-query]] 가 더 적합한 경우 많음.

### (E) Write-only atom (action)

```tsx
const countAtom = atom(0);

const incAtom = atom(
  null,                       // read: null
  (get, set) => set(countAtom, get(countAtom) + 1)
);

function Counter() {
  const [count] = useAtom(countAtom);
  const [, inc] = useAtom(incAtom);
  return <button onClick={inc}>{count}</button>;
}
```

## 4. useAtomValue / useSetAtom — 분리 hook

```tsx
import { useAtomValue, useSetAtom } from 'jotai';

// 값만 필요할 때
const count = useAtomValue(countAtom);

// setter 만 필요할 때 (값 구독 X → re-render 없음)
const setCount = useSetAtom(countAtom);
```

→ setter 만 쓰는 컴포넌트는 값 변경 시 re-render 안 함. **성능 최적화**.

## 5. 객체 / 배열 atom

```tsx
const userAtom = atom({ name: '', age: 0 });

// 업데이트
setUser({ ...user, age: 31 });

// or
setUser(prev => ({ ...prev, age: prev.age + 1 }));
```

→ useState 와 같은 immutable 규칙.

## 6. localStorage 영속

```tsx
import { atomWithStorage } from 'jotai/utils';

const themeAtom = atomWithStorage<'light' | 'dark'>('theme', 'light');

function ThemeToggle() {
  const [theme, setTheme] = useAtom(themeAtom);
  return (
    <button onClick={() => setTheme(t => t === 'light' ? 'dark' : 'light')}>
      {theme}
    </button>
  );
}
```

→ 새로고침에도 유지.

## 7. atomFamily — 동적 atom 생성

```tsx
import { atomFamily } from 'jotai/utils';

const todoAtomFamily = atomFamily((id: number) =>
  atom({ id, text: '', done: false })
);

function Todo({ id }: { id: number }) {
  const [todo, setTodo] = useAtom(todoAtomFamily(id));
  return <input value={todo.text} onChange={...} />;
}
```

→ list 의 각 row 가 자기만의 atom.

## 8. Provider — multi-store 또는 SSR

```tsx
import { Provider } from 'jotai';

// 일반: 안 써도 됨. global atom store 사용.

// 특정 subtree 에 별도 store:
<Provider>
  <SubApp />
</Provider>
```

→ test 시 격리된 atom store 만들기에 유용.

## 9. selectAtom — atom 의 일부만 구독

```tsx
import { selectAtom } from 'jotai/utils';

const userAtom = atom({ name: '유철', age: 30 });

const ageAtom = selectAtom(userAtom, (u) => u.age);

function Age() {
  const age = useAtomValue(ageAtom);
  // userAtom 의 age 만 바뀔 때 re-render. name 바뀌어도 영향 X.
}
```

## 10. answer-fe 의 흔한 패턴

```tsx
// store/userAtom.ts
import { atom } from 'jotai';
import { atomWithStorage } from 'jotai/utils';

export const tokenAtom = atomWithStorage<string | null>('token', null);
export const userAtom = atom<User | null>(null);
export const isLoggedInAtom = atom((get) => get(tokenAtom) !== null);

// LoginPage.tsx
const setToken = useSetAtom(tokenAtom);
const setUser = useSetAtom(userAtom);
await api.login(...).then(({ token, user }) => {
  setToken(token);
  setUser(user);
});

// Header.tsx
const isLoggedIn = useAtomValue(isLoggedInAtom);
const user = useAtomValue(userAtom);
```

## 11. recoil 와의 차이

| | recoil | jotai |
| --- | --- | --- |
| atom key | 명시 (`key: 'count'`) | 자동 (객체 참조) |
| Provider | 필수 `<RecoilRoot>` | optional |
| 유지보수 | Facebook 중단 | 활발 |
| async | atom.set(Promise) | atom(async () => ...) |
| Bundle | 8KB | 2.5KB |

→ **새 코드는 jotai 권장**.

### 마이그레이션 패턴
```tsx
// recoil
const countState = atom({ key: 'count', default: 0 });
const [count, setCount] = useRecoilState(countState);

// jotai
const countAtom = atom(0);
const [count, setCount] = useAtom(countAtom);
```

→ key 제거, 이름 ~Atom 으로 통일. API 거의 동일.

## 12. 함정

1. **새 atom 객체 매 render** — atom 은 컴포넌트 **밖에서** 정의 (전역 또는 module-level).
2. **거대 atom** — 1 atom 에 모든 state 넣으면 zustand 와 다를 바 없음. 작게 분리.
3. **async atom + react-query 혼용** — async atom 은 react-query 가 더 강력. 일반 데이터 fetching 은 react-query 권장.
4. **selectAtom 의 비교** — 기본 `Object.is`. deep equal 필요 시 `selectAtom(atom, fn, equalityFn)`.
5. **atomFamily 의 정리** — 동적 생성 atom 은 메모리 누적. `family.remove(id)` 또는 weakRef 모드.

## 13. 외부 자료

- [jotai 공식](https://jotai.org/)
- [recoil → jotai 마이그레이션](https://jotai.org/docs/recipes/recoil-migration)

## 14. 관련

- [[state-management]]
- [[recoil]]
- [[zustand]]
