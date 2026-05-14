---
title: "State Strategy — 어떤 상태를 어디에 둘까"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, state-management, architecture, decision-tree]
---

# State Strategy

**[[state-management|↑ State Management Hub]]**

> 새 상태를 만들 때 "어디 둘까" 의사결정 트리.

## 1. 상태의 5 종류 — 각 적합한 위치

| 종류 | 예 | 위치 |
| --- | --- | --- |
| **로컬 UI** | input 값, 토글, hover | useState |
| **공유 UI** | modal open, sidebar collapse | 부모 lift / context / zustand |
| **URL 상태** | 페이지, 필터, 정렬 | URL searchParams |
| **서버 상태** | API 응답, 사용자, 게시글 | react-query / SWR |
| **인증 / 세션** | 현재 사용자, token | context / zustand + 영속 storage |

## 2. 의사결정 트리

```
새 상태가 필요하다
        ↓
┌─────────────────────────┐
│ 서버에서 오는 데이터?    │ → YES → react-query / SWR
└─────────────────────────┘
        │ NO
        ↓
┌─────────────────────────┐
│ URL 에 담아도 되는가?   │ → YES → useSearchParams
│ (공유 / bookmark 가능?) │      (페이지, 필터, 정렬)
└─────────────────────────┘
        │ NO
        ↓
┌─────────────────────────┐
│ 한 컴포넌트만 쓰는가?    │ → YES → useState
└─────────────────────────┘
        │ NO
        ↓
┌─────────────────────────┐
│ 형제 / 가까운 컴포넌트?  │ → YES → lifting up to common parent
└─────────────────────────┘
        │ NO
        ↓
┌─────────────────────────┐
│ 변경 빈도 낮음 (테마)?  │ → YES → Context
└─────────────────────────┘
        │ NO
        ↓
        zustand / jotai (전역 store)
```

## 3. 각 단계의 깊은 이해

### (A) 서버 상태 — react-query 가 정답

"User 정보를 전역 store 에 저장" = 흔한 안티 패턴.

```tsx
// ❌ 옛 방식
function App() {
  const [user, setUser] = useState(null);
  useEffect(() => {
    fetch('/api/me').then(r => r.json()).then(setUser);
  }, []);
  return <UserContext.Provider value={user}>...</UserContext.Provider>;
}

// ✅ react-query
function App() {
  // App 어디서나
}

function useMe() {
  return useQuery({ queryKey: ['me'], queryFn: () => fetch('/api/me').then(r => r.json()) });
}
```

→ react-query 가 **캐싱 / refetch / loading / error / stale** 다 해결. [[../server-state/react-query]].

### (B) URL 상태 — 공유 / 새로고침 가능

```tsx
// 필터 / 페이지를 URL 에
// /products?category=shoes&page=2

const [params, setParams] = useSearchParams();
const category = params.get('category');
const page = Number(params.get('page') ?? 1);

setParams({ category, page: String(page + 1) });
```

장점:
- 사용자가 URL 공유 → 같은 화면 재현.
- 새로고침에도 상태 유지.
- 브라우저 뒤로가기로 자연 복원.

UI 상태로 검색 form 의 값, 정렬, 페이지, 탭 등 → **가급적 URL**.

### (C) 로컬 useState — 1 컴포넌트

```tsx
// modal 안 임시 입력
const [draft, setDraft] = useState('');

// 토글
const [isExpanded, setIsExpanded] = useState(false);
```

### (D) Lifting up — 형제 공유

```tsx
function Parent() {
  const [selected, setSelected] = useState<number | null>(null);
  return (
    <>
      <List onSelect={setSelected} />
      <Detail selectedId={selected} />
    </>
  );
}
```

### (E) Context — 자주 변경 안 됨

- 테마.
- 로케일.
- 인증 사용자 (변경 거의 X).
- design system 의 size / variant.

자주 변경 시 → 모든 소비자 re-render. zustand 권장.

### (F) 전역 store — 그래도 필요할 때

- 다크 모드 토글.
- 전역 알림 (toast).
- 장바구니 (URL 부적합 + 여러 페이지).
- 채팅 unread 카운트.

## 4. 안티 패턴 — 흔한 실수

### (1) 모든 것을 전역 store 로
```tsx
// ❌ modal 도 toast 도 input 도 다 redux store 로
dispatch({ type: 'SET_INPUT_VALUE', payload: e.target.value });
```
→ 로컬 state 가 가장 단순. 라이브러리 거치며 비용만 증가.

### (2) 서버 상태를 store 에 직접
```tsx
// ❌
const [users, setUsers] = useState([]);
useEffect(() => { fetch('/users').then(r => r.json()).then(setUsers); }, []);

// 그리고 다른 컴포넌트에서 같은 fetch... 또는 globalStore.users 에 저장
```
→ react-query 가 같은 queryKey 면 1 회만 fetch + 캐시 공유.

### (3) Derived state 를 별도 state 로
```tsx
// ❌
const [items, setItems] = useState([1, 2, 3]);
const [count, setCount] = useState(3);  // 별도 관리 → 동기화 버그

// ✅
const [items, setItems] = useState([1, 2, 3]);
const count = items.length;             // derive
```

### (4) Context 의 거대 객체
```tsx
// ❌
<AppContext.Provider value={{ user, settings, notifications, theme, ... }}>
```
→ 한 값 바뀌면 전체 re-render. **Context 분리** 또는 zustand.

## 5. 실전 매핑 — 예시 사례

| 상태 | 어디 |
| --- | --- |
| 로그인 form input | useState (Form 컴포넌트) |
| 로그인 응답 (user, token) | react-query (`['me']`) + token 은 localStorage / cookie |
| 페이지 번호 | URL `?page=2` |
| 검색어 | URL `?q=react` |
| modal open / close | useState (그 페이지) 또는 zustand (전역 modal manager) |
| 다크 모드 | Context (자주 안 바뀜) 또는 zustand + localStorage |
| 게시글 목록 | react-query |
| 게시글 작성 form | useState 또는 react-hook-form |
| 장바구니 | zustand + localStorage 영속 |
| 알림 (toast) | zustand store (전역에서 호출) |
| 사이드바 펼침 여부 | localStorage + zustand |
| 채팅 메시지 | react-query (서버) + ws subscription 으로 invalidate |

## 6. 영속화 — localStorage 와 통합

```tsx
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

const useSettings = create(
  persist(
    (set) => ({
      theme: 'light',
      setTheme: (t) => set({ theme: t }),
    }),
    { name: 'settings' }
  )
);
```

→ 새로고침 후에도 유지.

## 7. masterway-dev 의 패턴 추정

```bash
# 어떤 store 가 정의돼 있나
ls ~/masterway-dev/answer-fe/src/store/
ls ~/masterway-dev/job-answer-fe/src/store/ 2>/dev/null
```

전형:
- `userStore.ts` (recoil) — 현재 로그인 사용자.
- `modalStore.ts` — 전역 modal.
- `toastStore.ts` — 알림.

→ 보면서 위 매핑 적용해보기.

## 8. 정리 — 1 줄 원칙

> **"상태는 사용자에게 가장 가깝게 두기."**
> 1 컴포넌트면 거기. 형제면 부모. 페이지 사이면 URL. 그래도 안 되면 store.

## 9. 다음 단계

- [[jotai]] / [[zustand]] / [[recoil]] — 각 라이브러리
- [[../server-state/react-query]] — 서버 상태

## 10. 관련

- [[state-management]]
- [[../server-state/server-state]]
- [[../core-concepts/props-and-state]]
