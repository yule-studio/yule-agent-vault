---
title: "State Management — 어떤 상태를 어디에 둘까"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, state-management, jotai, recoil, zustand, hub]
---

# State Management

**[[../react|↑ React Hub]]**

> "전역 상태" 라이브러리 선택은 코드 스타일 결정. **선택지: jotai / recoil / zustand / redux / context**.

## 1. 우선 — 정말 전역이 필요한가?

상태는 **가능한 한 가장 좁은 scope** 에 두기.

```
1. 한 컴포넌트만 쓰면 → useState (그 컴포넌트)
2. 부모-자식 → props
3. 형제 → 공통 부모로 lifting up
4. 깊은 props drilling → context 또는 전역 store
5. 전혀 다른 페이지 사이 공유 → 전역 store
```

→ 처음부터 모든 state 를 전역으로 = 안티 패턴.

## 2. 서버 상태 vs 클라이언트 상태 — 구분

| | 예 |
| --- | --- |
| **클라이언트 상태** | UI 토글, modal open, 폼 입력, 다크 모드 |
| **서버 상태** | API 응답, 사용자 정보, 게시글 목록 |

→ **서버 상태는 react-query / SWR 의 영역**. jotai / recoil / zustand 는 클라이언트 상태.

자세히 [[../server-state/server-state]].

## 3. 선택지 비교

| | bundle | API | 학습 | 추천 |
| --- | --- | --- | --- | --- |
| **useState + Context** | 0 | 표준 React | 쉬움 | 작은 앱 |
| **zustand** | 1.5KB | 간단 (set/get) | 매우 쉬움 | 가벼운 전역 |
| **jotai** | 2.5KB | atom | 쉬움 | atomic, fine-grained |
| **recoil** | 8KB | atom + selector | 쉬움 | atomic, Facebook |
| **redux + RTK** | 13KB | reducer | 어려움 | 대규모, 미들웨어 |
| **mobx** | 16KB | observable | 보통 | reactive 모델 선호 |

`masterway-dev/answer-fe` = **jotai + recoil 혼용** (역사적 이유).
`masterway-dev/job-answer-fe` = **recoil** 단독.

## 4. 자기 프로젝트 확인

```bash
grep -E "jotai|recoil|zustand|redux|mobx" ~/masterway-dev/answer-fe/package.json
grep -E "jotai|recoil|zustand|redux|mobx" ~/masterway-dev/job-answer-fe/package.json
```

## 5. zustand — 가장 간단

```tsx
import { create } from 'zustand';

interface BearStore {
  count: number;
  increase: () => void;
  reset: () => void;
}

const useBear = create<BearStore>((set) => ({
  count: 0,
  increase: () => set(s => ({ count: s.count + 1 })),
  reset: () => set({ count: 0 }),
}));

// 사용
function Counter() {
  const count = useBear(s => s.count);
  const increase = useBear(s => s.increase);
  return <button onClick={increase}>{count}</button>;
}
```

→ Provider 안 씀. import 만 하면 끝. [[zustand]] 자세히.

## 6. jotai — Atom 단위

```tsx
import { atom, useAtom } from 'jotai';

const countAtom = atom(0);

function Counter() {
  const [count, setCount] = useAtom(countAtom);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

→ atom = useState 처럼 단순. 어디서나 같은 atom 공유. [[jotai]] 자세히.

## 7. recoil — Atom + Selector

```tsx
import { atom, useRecoilState, selector, useRecoilValue } from 'recoil';

const countState = atom({ key: 'count', default: 0 });
const doubleState = selector({
  key: 'double',
  get: ({ get }) => get(countState) * 2,
});

function Counter() {
  const [count, setCount] = useRecoilState(countState);
  const double = useRecoilValue(doubleState);
  return (
    <>
      <p>{count} (x2 = {double})</p>
      <button onClick={() => setCount(c => c + 1)}>+1</button>
    </>
  );
}
```

→ atom + 파생 (selector) 강력. **Facebook 유지보수 중단됨 → jotai 권장**. [[recoil]] 자세히.

## 8. Context — React 내장

```tsx
const ThemeContext = createContext<'light' | 'dark'>('light');

function App() {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  return (
    <ThemeContext.Provider value={theme}>
      <Page />
    </ThemeContext.Provider>
  );
}

function Page() {
  const theme = useContext(ThemeContext);
  return <p>{theme}</p>;
}
```

→ 변경 빈도 낮은 값 (테마, locale, 인증 사용자) 에 적합.
변경 잦으면 → context 안 모든 소비자 re-render 폭주. zustand / jotai 사용.

## 9. Redux Toolkit (RTK)

```tsx
import { configureStore, createSlice } from '@reduxjs/toolkit';

const counter = createSlice({
  name: 'counter',
  initialState: { value: 0 },
  reducers: {
    inc: (state) => { state.value += 1; },  // immer 로 mutate 가능
  },
});

const store = configureStore({ reducer: { counter: counter.reducer } });

// 사용
import { useDispatch, useSelector } from 'react-redux';
const value = useSelector(s => s.counter.value);
const dispatch = useDispatch();
dispatch(counter.actions.inc());
```

→ 대규모 / 미들웨어 (saga / thunk) / devtools 강력. **boilerplate 많음**. `masterway-dev/*-fe` 에는 사용 X.

## 10. 선택 가이드 — 새 프로젝트라면

```
프로젝트 시작
  ↓
useState + 가끔 context 로 시작
  ↓
state 공유 늘면 →
  ↓
↘ 단순 전역 = zustand
↘ atomic = jotai
↘ 대규모 / 팀 / 미들웨어 = Redux Toolkit
```

→ 2025+ 트렌드: **jotai** 또는 **zustand** 가 권장. Redux 의 boilerplate 부담.

## 11. 기존 프로젝트라면

- recoil 사용 → **점진적으로 jotai 이전** (API 유사).
- redux 사용 → 한꺼번에 교체 X. 새 기능부터 zustand/jotai.
- jotai 사용 → 유지.

## 12. 학습 순서

1. **[[state-strategy]]** — "어떤 상태를 어디에" 의사 결정 트리.
2. **[[zustand]]** — 가장 단순. 1 일.
3. **[[jotai]]** — recoil 의 후속. 2 일.
4. **[[recoil]]** — `masterway-dev` 의 기존 코드 이해용.
5. **[[../server-state/react-query]]** — 서버 상태는 따로.

## 13. 함정

1. **서버 상태를 전역 store 에** — react-query 가 캐싱 / refetch 무료 제공.
2. **모든 state 를 전역** — local 로 시작. 진짜 공유 필요할 때만.
3. **Context 의 빈번 변경** — 소비자 re-render. zustand 로 이전.
4. **Store 안 mutation** — zustand / jotai 도 immutable update. RTK 만 immer 자동.
5. **store 직접 import** — 큰 store 의 일부만 구독 권장 (selector).
6. **Provider 누락** — recoil 의 `<RecoilRoot>`, redux 의 `<Provider>`. jotai 는 optional, zustand 는 불필요.

## 14. 외부 자료

- [zustand 공식](https://zustand-demo.pmnd.rs/)
- [jotai 공식](https://jotai.org/)
- [TkDodo "Why I don't use Redux anymore"](https://tkdodo.eu/blog/react-query-as-a-state-manager) (react-query 와 함께 read)

## 15. 관련

- [[state-strategy]]
- [[jotai]] / [[recoil]] / [[zustand]]
- [[../server-state/server-state]]
