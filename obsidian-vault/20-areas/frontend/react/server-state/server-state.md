---
title: "Server State — 서버에서 오는 데이터 다루기"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, server-state, react-query, hub, beginner]
---

# Server State

**[[../react|↑ React Hub]]**

> 서버 데이터 = 캐싱 / refetch / loading / error / 동기화의 영역. **react-query 가 정답**.

## 1. 왜 서버 상태가 특별한가?

```tsx
// 클라이언트 상태 (다크 모드)
- 내가 만든 값. 내가 변경. 변경되면 끝.

// 서버 상태 (사용자 목록)
- 서버에 진실. 내 copy 는 stale 가능.
- 다른 사용자가 변경 가능.
- 새로고침 시 다시 받아야.
- loading / error 상태 처리 필요.
- 같은 데이터를 여러 페이지에서 fetch → 캐싱 필요.
```

→ **useState + useEffect 로 fetch 하는 것 = 안티 패턴**. 캐싱 / refetch / dedup 없음.

## 2. 라이브러리 선택

| | 특징 |
| --- | --- |
| **TanStack Query (react-query)** | 사실상 표준. v3 → v4 → v5 발전. |
| **SWR** | Vercel 제작. 더 가벼움. |
| **RTK Query** | Redux Toolkit 쓰면 자연스러움. |
| **Apollo Client** | GraphQL 전용. |

→ 학습 우선: **react-query**. [[react-query]] 자세히.

`masterway-dev/answer-fe` = react-query v4.
`masterway-dev/job-answer-fe` = react-query v3.

## 3. react-query 의 핵심 4 가지

### (A) useQuery — 데이터 fetch + 캐싱

```tsx
import { useQuery } from '@tanstack/react-query';

function UserList() {
  const { data, isLoading, error } = useQuery({
    queryKey: ['users'],
    queryFn: () => fetch('/api/users').then(r => r.json()),
  });

  if (isLoading) return <Spinner />;
  if (error) return <Error />;
  return <ul>{data.map(...)}</ul>;
}
```

### (B) useMutation — 변경 (POST/PUT/DELETE)

```tsx
import { useMutation, useQueryClient } from '@tanstack/react-query';

function AddUser() {
  const qc = useQueryClient();
  const mutation = useMutation({
    mutationFn: (user) => fetch('/api/users', { method: 'POST', body: JSON.stringify(user) }),
    onSuccess: () => qc.invalidateQueries({ queryKey: ['users'] }),  // 자동 refetch
  });

  return <button onClick={() => mutation.mutate({ name: '새 사용자' })}>추가</button>;
}
```

### (C) Cache invalidation

```tsx
qc.invalidateQueries({ queryKey: ['users'] });   // users 캐시 무효화 → 화면 갱신
```

### (D) staleTime / cacheTime — 캐시 정책

```tsx
useQuery({
  queryKey: ['users'],
  queryFn: ...,
  staleTime: 60 * 1000,        // 1 분은 fresh — refetch 안 함
  gcTime: 5 * 60 * 1000,       // 5 분간 unmount 후 cache 보존 (v5: gcTime)
});
```

자세히 [[react-query]].

## 4. 서버 상태를 전역 store 에 두지 마라

```tsx
// ❌ 옛 패턴
const [users, setUsers] = useState([]);
useEffect(() => { fetch('/users').then(r => r.json()).then(setUsers); }, []);
// 또는 store 에 users 저장

// 안 되는 점:
// - 캐싱 X — 페이지 진입할 때마다 fetch.
// - 동시 두 컴포넌트 같은 데이터 fetch — 중복 호출.
// - mutation 후 수동 동기화.
// - loading / error 수동 관리.
// - background refetch 없음.
```

```tsx
// ✅ react-query
const { data: users } = useQuery({ queryKey: ['users'], queryFn: getUsers });
// → 같은 queryKey 의 다른 컴포넌트와 cache 공유.
// → mutation 시 invalidate 만 하면 자동 refetch.
// → window focus 시 자동 refetch.
```

## 5. 학습 우선순위

1. **[[react-query]]** — useQuery / useMutation / invalidateQueries.
2. **queryKey 전략** — 어떻게 이름 붙일까.
3. **staleTime / refetchOnWindowFocus** — 캐시 정책.
4. **optimistic update** — UI 먼저 변경, 실패 시 revert.
5. **infinite query / suspense** — 응용.

## 6. masterway-dev 패턴 확인

```bash
grep -rn "useQuery\|useMutation" ~/masterway-dev/answer-fe/src --include="*.tsx" | head -10
```

전형:
```tsx
// hooks/useUser.ts
export function useUser(id: number) {
  return useQuery({
    queryKey: ['user', id],
    queryFn: () => api.getUser(id),
    enabled: !!id,
  });
}

// 사용
function UserPage() {
  const { id } = useParams();
  const { data: user, isLoading } = useUser(Number(id));
  ...
}
```

## 7. 함정

1. **queryKey 일관성** — `['user', id]` vs `['users', id]` 헷갈리면 캐시 무효화 어긋남.
2. **useEffect 안 useQuery 호출** — hook 규칙 위반. top-level 만.
3. **stale data 가정** — `data` 가 마지막 성공 데이터. mutation 직후엔 옛 값 일 수 있음 (refetch 전까지).
4. **enabled 옵션** — `useQuery({ enabled: !!userId })` 로 조건부 fetch.
5. **API error 처리** — react-query 는 `throw` 한 error 만 잡음. fetch 는 4xx/5xx 도 resolve. axios 권장 또는 수동 throw.
6. **Provider 누락** — `<QueryClientProvider client={qc}>` 가 root 에.

## 8. 외부 자료

- [TanStack Query 공식](https://tanstack.com/query/latest)
- [TkDodo's Practical React Query](https://tkdodo.eu/blog/practical-react-query) — 거의 필독.

## 9. 관련

- [[react-query]]
- [[../state-management/state-management]]
- [[../http/axios-setup]]
