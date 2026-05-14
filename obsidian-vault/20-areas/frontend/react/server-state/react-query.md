---
title: "TanStack Query (react-query) — 실전"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, react-query, tanstack-query, server-state]
---

# TanStack Query (react-query)

**[[server-state|↑ Server State Hub]]**

> v3 → v4 → v5 거치며 API 변화. **`masterway-dev/answer-fe = v4`**, **`job-answer-fe = v3`**. 신규는 v5 권장.

## 1. 설치 + Setup

```bash
yarn add @tanstack/react-query        # v4 / v5
# v3 은 yarn add react-query (구 패키지)
```

```tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60 * 1000,
      retry: 1,
    },
  },
});

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <BrowserRouter>...</BrowserRouter>
    </QueryClientProvider>
  );
}
```

## 2. useQuery — 기본

```tsx
import { useQuery } from '@tanstack/react-query';

function UserList() {
  const { data, isLoading, isError, error, refetch } = useQuery({
    queryKey: ['users'],
    queryFn: async () => {
      const r = await axios.get('/api/users');
      return r.data;
    },
  });

  if (isLoading) return <Spinner />;
  if (isError) return <p>에러: {error.message}</p>;
  return <ul>{data.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

### 반환 값

| | 의미 |
| --- | --- |
| `data` | 마지막 성공 데이터 |
| `isLoading` | 첫 fetch 중 (v5: `isPending`) |
| `isFetching` | 백그라운드 fetch 포함 |
| `isError` | 에러 발생 |
| `error` | 에러 객체 |
| `refetch` | 수동 refetch 함수 |
| `isSuccess` | 성공 |

## 3. queryKey 설계

```tsx
// 단순
['users']

// 파라미터 포함
['user', userId]
['users', { page, sort, filter }]

// 도메인-자원-id 패턴 권장
['questions']
['questions', questionId]
['questions', questionId, 'answers']
```

→ **객체는 deep equal 비교**. 같은 데이터면 같은 키.

### Type-safe queryKey factory

```tsx
const userKeys = {
  all: ['users'] as const,
  detail: (id: number) => [...userKeys.all, id] as const,
  list: (filter: Filter) => [...userKeys.all, 'list', filter] as const,
};

useQuery({ queryKey: userKeys.detail(5), queryFn: () => api.getUser(5) });
```

→ invalidate 시 일관성. 대규모 권장.

## 4. queryFn 의 패턴

```tsx
// (1) inline
useQuery({
  queryKey: ['user', id],
  queryFn: () => fetch(`/users/${id}`).then(r => r.json()),
});

// (2) axios
useQuery({
  queryKey: ['user', id],
  queryFn: async () => {
    const r = await axios.get(`/users/${id}`);
    return r.data;
  },
});

// (3) signal (취소 지원)
useQuery({
  queryKey: ['user', id],
  queryFn: ({ signal }) => axios.get(`/users/${id}`, { signal }).then(r => r.data),
});
```

→ react-query 가 unmount 시 자동 cancel 가능.

## 5. enabled — 조건부 fetch

```tsx
const { data } = useQuery({
  queryKey: ['user', userId],
  queryFn: () => api.getUser(userId!),
  enabled: !!userId,    // userId 있을 때만 fetch
});
```

→ 로그인 후 fetch, params 의존 fetch 등.

## 6. staleTime / gcTime — 캐시 정책

```tsx
useQuery({
  queryKey: ['users'],
  queryFn: ...,
  staleTime: 60 * 1000,        // 1 분은 fresh
  gcTime: 5 * 60 * 1000,        // 5 분간 unused cache 보존 (v4 cacheTime)
  refetchOnWindowFocus: true,   // 창 focus 시 (stale 면) refetch
  refetchOnMount: true,
  refetchOnReconnect: true,
});
```

| | 의미 |
| --- | --- |
| `staleTime` | "fresh" 로 간주하는 시간. fresh = refetch 안 함. |
| `gcTime` (v5) / `cacheTime` (v4) | cache 가 메모리에 머무는 시간. unmount 후 카운트. |

### 정책 예
- **거의 안 바뀜 (설정, 사용자 본인)**: `staleTime: Infinity` 또는 매우 김.
- **자주 바뀜 (실시간 게시판)**: `staleTime: 0` + `refetchInterval`.

## 7. useMutation — 변경

```tsx
import { useMutation, useQueryClient } from '@tanstack/react-query';

function CreateUser() {
  const qc = useQueryClient();

  const mutation = useMutation({
    mutationFn: (data: { name: string }) =>
      axios.post('/users', data).then(r => r.data),
    onSuccess: (newUser) => {
      qc.invalidateQueries({ queryKey: ['users'] });   // 목록 refetch
      // 또는 직접 cache update
      qc.setQueryData(['user', newUser.id], newUser);
    },
    onError: (error) => {
      toast.error(error.message);
    },
  });

  return (
    <button onClick={() => mutation.mutate({ name: '새 사용자' })}>
      추가 {mutation.isPending && '...'}
    </button>
  );
}
```

### mutation 의 반환

| | 의미 |
| --- | --- |
| `mutate(vars)` | trigger |
| `mutateAsync(vars)` | Promise return |
| `isPending` (v5) / `isLoading` (v4) | 진행 중 |
| `isError` / `error` | 에러 |
| `isSuccess` / `data` | 성공 |

## 8. invalidate vs setQueryData

```tsx
// invalidate — 다음 사용 시 refetch
qc.invalidateQueries({ queryKey: ['users'] });

// setQueryData — 즉시 cache 업데이트 (refetch 없음)
qc.setQueryData(['user', id], updatedUser);

// 둘 다 — 즉시 화면 갱신 + 백그라운드 refetch 로 검증
qc.setQueryData(['user', id], updatedUser);
qc.invalidateQueries({ queryKey: ['user', id] });
```

## 9. Optimistic update — UI 먼저 변경

```tsx
useMutation({
  mutationFn: updateUser,
  onMutate: async (newUser) => {
    await qc.cancelQueries({ queryKey: ['user', newUser.id] });
    const previous = qc.getQueryData(['user', newUser.id]);
    qc.setQueryData(['user', newUser.id], newUser);
    return { previous };       // context
  },
  onError: (err, newUser, context) => {
    qc.setQueryData(['user', newUser.id], context?.previous);   // 롤백
  },
  onSettled: () => qc.invalidateQueries({ queryKey: ['user'] }),
});
```

## 10. useInfiniteQuery — 무한 스크롤

```tsx
const { data, fetchNextPage, hasNextPage } = useInfiniteQuery({
  queryKey: ['posts'],
  queryFn: ({ pageParam = 0 }) => axios.get(`/posts?page=${pageParam}`).then(r => r.data),
  initialPageParam: 0,    // v5
  getNextPageParam: (lastPage) => lastPage.nextPage,
});

// 화면
{data?.pages.map((page) => page.items.map((item) => <Card key={item.id} item={item} />))}
<button onClick={() => fetchNextPage()} disabled={!hasNextPage}>더 보기</button>
```

→ [[../recipes/infinite-scroll]] 에서 자세히.

## 11. Suspense 모드

```tsx
useSuspenseQuery({
  queryKey: ['user', id],
  queryFn: () => api.getUser(id),
});
// → loading 상태 없음. Suspense fallback 으로.

<Suspense fallback={<Loading />}>
  <UserPage />
</Suspense>
```

→ v5 의 새 API. 깔끔.

## 12. v3 → v4 → v5 변화 요약

| | v3 | v4 | v5 |
| --- | --- | --- | --- |
| import | `react-query` | `@tanstack/react-query` | 동일 |
| signature | `useQuery(key, fn)` | `useQuery({ queryKey, queryFn })` | 동일 |
| isLoading | first load | first load | `isPending` 권장 |
| cacheTime | `cacheTime` | `cacheTime` | `gcTime` |
| keepPrevious | 옵션 | 옵션 | `placeholderData: keepPreviousData` |

→ v4 가 큰 변화. v4→v5 는 점진적.

## 13. answer-fe 패턴 (v4)

```tsx
// hooks/useUser.ts
export const useUser = (id?: number) => {
  return useQuery({
    queryKey: ['user', id],
    queryFn: () => api.getUser(id!),
    enabled: !!id,
    staleTime: 60_000,
  });
};

// hooks/useUpdateUser.ts
export const useUpdateUser = () => {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: api.updateUser,
    onSuccess: (_, vars) => {
      qc.invalidateQueries({ queryKey: ['user', vars.id] });
    },
  });
};
```

→ custom hook 으로 wrap 후 page 에서 import.

## 14. 함정

1. **queryKey 의 변화 → 새 fetch** — params 객체를 매번 새로 만들면 매 render 새 fetch. memoize.
2. **fetch 의 4xx/5xx 가 resolve** — fetch 는 throw 안 함. `if (!r.ok) throw new Error(...)`. axios 는 자동 throw.
3. **invalidate 의 부분 match** — `['user']` invalidate 면 `['user', 1]`, `['user', 2]` 다 무효. 의도 확인.
4. **mutation 의 stale closure** — `onSuccess` 안 closure 에 옛 state. 함수형 update.
5. **Suspense + Error Boundary** — Suspense 만 쓰면 error 가 위로. ErrorBoundary 필수.
6. **dev 모드 useEffect 2 번 mount** — Strict Mode 가 useQuery 도 2 번 호출. dedup 되긴 하지만 mock 환경에선 혼란.
7. **Provider 누락** — `<QueryClientProvider>` 없으면 throw.

## 15. 외부 자료

- [TanStack Query 공식 v5](https://tanstack.com/query/v5)
- [TkDodo Practical React Query 시리즈](https://tkdodo.eu/blog/practical-react-query)

## 16. 관련

- [[server-state]]
- [[../http/axios-setup]]
- [[../recipes/infinite-scroll]]
- [[../state-management/state-strategy]]
