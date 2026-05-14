---
title: "zustand — 가장 단순한 전역 store"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, state-management, zustand]
---

# zustand

**[[state-management|↑ State Management Hub]]**

> 1.5KB. Provider 없음. import 만 하면 끝. 신규 프로젝트에 강력 추천.

## 1. 설치

```bash
yarn add zustand
```

## 2. 최소 예

```tsx
import { create } from 'zustand';

interface BearStore {
  count: number;
  inc: () => void;
}

const useBear = create<BearStore>((set) => ({
  count: 0,
  inc: () => set((s) => ({ count: s.count + 1 })),
}));

function Counter() {
  const count = useBear((s) => s.count);
  const inc = useBear((s) => s.inc);
  return <button onClick={inc}>{count}</button>;
}
```

→ `create((set) => ({ ... }))` 가 store. hook (`useBear`) 으로 selector 구독.

## 3. selector 패턴 — 일부만 구독

```tsx
// ❌ 전체 구독 — 어느 field 든 바뀌면 re-render
const store = useBear();

// ✅ 일부만 구독
const count = useBear((s) => s.count);
const inc = useBear((s) => s.inc);

// 여러 값 — shallow
import { shallow } from 'zustand/shallow';
const { count, name } = useBear((s) => ({ count: s.count, name: s.name }), shallow);
```

→ shallow 가 없으면 객체 새로 만들 때마다 re-render.

## 4. 비동기 action

```tsx
interface UserStore {
  user: User | null;
  isLoading: boolean;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
}

const useUser = create<UserStore>((set) => ({
  user: null,
  isLoading: false,

  login: async (email, password) => {
    set({ isLoading: true });
    try {
      const user = await api.login(email, password);
      set({ user, isLoading: false });
    } catch (e) {
      set({ isLoading: false });
      throw e;
    }
  },

  logout: () => set({ user: null }),
}));
```

→ async/await 그대로. 어떤 미들웨어도 필요 없음.

## 5. localStorage 영속 — persist 미들웨어

```tsx
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

const useSettings = create(
  persist<{ theme: 'light' | 'dark'; setTheme: (t: 'light' | 'dark') => void }>(
    (set) => ({
      theme: 'light',
      setTheme: (t) => set({ theme: t }),
    }),
    {
      name: 'settings',                                  // localStorage key
      // storage: createJSONStorage(() => sessionStorage), // sessionStorage 로
    }
  )
);
```

→ 새로고침에도 유지.

## 6. devtools 미들웨어 — Redux DevTools 와 통합

```tsx
import { devtools } from 'zustand/middleware';

const useBear = create(devtools<BearStore>((set) => ({ ... }), { name: 'bear-store' }));
```

## 7. immer 미들웨어 — mutation 처럼 작성

```tsx
import { immer } from 'zustand/middleware/immer';

const useTodo = create(immer<TodoStore>((set) => ({
  todos: [],
  add: (text) => set((state) => {
    state.todos.push({ text, done: false });     // mutate OK (immer)
  }),
  toggle: (i) => set((state) => {
    state.todos[i].done = !state.todos[i].done;
  }),
})));
```

→ 객체 / 배열 깊은 update 가 깔끔.

## 8. store 분할 + 합성

```tsx
const useBearStore = create<BearStore>((set) => ({ ... }));
const useFishStore = create<FishStore>((set) => ({ ... }));

// 또는 한 store 안 slice 패턴
const useStore = create<Store>((set, get) => ({
  ...createBearSlice(set, get),
  ...createFishSlice(set, get),
}));
```

→ 도메인별 store 가 가장 단순.

## 9. computed (selector) — derive

```tsx
const useTodo = create<TodoStore>((set, get) => ({
  todos: [],
  add: (t) => set({ todos: [...get().todos, t] }),
}));

// 컴포넌트에서 derive
function TodoCount() {
  const count = useTodo((s) => s.todos.length);
  return <p>{count}</p>;
}

function UndoneCount() {
  const undone = useTodo((s) => s.todos.filter(t => !t.done).length);
  return <p>{undone}</p>;
}
```

## 10. store 밖에서 직접 사용 — utility 함수

```tsx
// store
const useUser = create<UserStore>(...);

// 컴포넌트 밖 (예: axios interceptor)
import { useUser } from './store/user';

api.interceptors.request.use((config) => {
  const token = useUser.getState().token;     // hook 아닌 getState
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// 또는 직접 set
useUser.setState({ user: null });
```

→ hook 못 쓰는 곳에서.

## 11. SSR / Next.js — Provider 패턴

(필요 시) Next.js 의 SSR 에서 hydration mismatch 방지:
```tsx
import { useStore } from 'zustand';

const initStore = () => create<Store>(...);

// 매 SSR request 마다 새 store
```

→ CSR-only 인 `masterway-dev` 의 -fe 는 신경 안 써도 됨.

## 12. zustand vs jotai vs recoil

| | zustand | jotai | recoil |
| --- | --- | --- | --- |
| 단위 | store (객체) | atom (값) | atom + key |
| 학습 | 매우 쉬움 | 쉬움 | 쉬움 |
| bundle | 1.5KB | 2.5KB | 8KB |
| Provider | 없음 | optional | 필수 |
| 적합 | 도메인별 store | 작은 값 | (deprecated) |

→ "한 도메인의 여러 값 + action" = zustand. "독립된 작은 값" = jotai.

## 13. 함정

1. **store 객체 전체 구독** — selector 안 쓰면 어디 바뀌어도 re-render.
2. **새 객체 selector 매 호출** — `s => ({ a, b })` 는 매번 새 객체. `shallow` 사용.
3. **localStorage 직렬화 — function 안 됨** — action 은 자동 제외. 값만 저장.
4. **state 직접 mutate** — `set((s) => { s.count += 1; return s })` ❌. immer 미들웨어 또는 spread.
5. **deep nested update** — `set({ user: { ...user, profile: { ...user.profile, age: 31 } } })` 지저분. immer 권장.
6. **store 안에서 다른 store 호출** — `useOther.getState()` 로 (hook 아닌). 단 순환 의존 주의.

## 14. 외부 자료

- [zustand 공식](https://zustand-demo.pmnd.rs/)
- [TkDodo "Working with Zustand"](https://tkdodo.eu/blog/working-with-zustand)

## 15. 관련

- [[state-management]]
- [[jotai]]
- [[../server-state/server-state]]
