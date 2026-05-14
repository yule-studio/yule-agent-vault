---
title: "Infinite Scroll — IntersectionObserver + useInfiniteQuery"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, recipes, infinite-scroll, react-query, intersection-observer]
---

# Infinite Scroll

**[[recipes|↑ Recipes Hub]]**

> 스크롤 끝에 도달하면 자동 다음 페이지. **react-query 의 useInfiniteQuery + IntersectionObserver**.

## 1. 큰 그림

```
페이지 1 로딩 → 사용자 스크롤 → 마지막 row 가 보임
       ↓ (IntersectionObserver 가 감지)
페이지 2 자동 fetch → append
       ↓
... 반복
```

## 2. useInfiniteQuery — 데이터 부분

```tsx
import { useInfiniteQuery } from '@tanstack/react-query';

const fetchPosts = async ({ pageParam = 0 }) => {
  const { data } = await axios.get(`/posts?page=${pageParam}&size=20`);
  return data;   // { items: [...], nextPage: number | null }
};

function PostList() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isLoading,
  } = useInfiniteQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
    initialPageParam: 0,
    getNextPageParam: (lastPage) => lastPage.nextPage,
  });

  if (isLoading) return <Spinner />;

  return (
    <>
      <ul>
        {data?.pages.flatMap(p => p.items).map(item => (
          <li key={item.id}>{item.title}</li>
        ))}
      </ul>
      {isFetchingNextPage && <Spinner />}
      {!hasNextPage && <p>끝</p>}
    </>
  );
}
```

→ `data.pages` 가 page 배열. `flatMap` 으로 평탄화.

## 3. trigger — IntersectionObserver

### 방법 1 — useRef + 직접 observer

```tsx
import { useEffect, useRef } from 'react';

const loadMoreRef = useRef<HTMLDivElement>(null);

useEffect(() => {
  if (!loadMoreRef.current || !hasNextPage) return;

  const observer = new IntersectionObserver((entries) => {
    if (entries[0].isIntersecting && !isFetchingNextPage) {
      fetchNextPage();
    }
  }, { threshold: 0.1 });

  observer.observe(loadMoreRef.current);
  return () => observer.disconnect();
}, [hasNextPage, isFetchingNextPage, fetchNextPage]);

return (
  <>
    <ul>...</ul>
    <div ref={loadMoreRef} style={{ height: 20 }} />     {/* trigger element */}
  </>
);
```

→ 마지막에 보이지 않는 div. 화면에 들어오면 다음 page fetch.

### 방법 2 — `react-intersection-observer` 라이브러리

```bash
yarn add react-intersection-observer
```

```tsx
import { useInView } from 'react-intersection-observer';

function PostList() {
  const { ref, inView } = useInView();
  const { fetchNextPage, hasNextPage, isFetchingNextPage, data } = useInfiniteQuery(...);

  useEffect(() => {
    if (inView && hasNextPage && !isFetchingNextPage) {
      fetchNextPage();
    }
  }, [inView, hasNextPage, isFetchingNextPage]);

  return (
    <>
      <ul>...</ul>
      <div ref={ref}>{isFetchingNextPage && <Spinner />}</div>
    </>
  );
}
```

→ 더 간결.

## 4. 완성 컴포넌트

```tsx
import { useInfiniteQuery } from '@tanstack/react-query';
import { useInView } from 'react-intersection-observer';
import { useEffect } from 'react';

function PostList() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isLoading,
    error,
  } = useInfiniteQuery({
    queryKey: ['posts'],
    queryFn: ({ pageParam = 0 }) => api.get(`/posts?page=${pageParam}`).then(r => r.data),
    initialPageParam: 0,
    getNextPageParam: (last) => last.nextPage,
  });

  const { ref, inView } = useInView();

  useEffect(() => {
    if (inView && hasNextPage && !isFetchingNextPage) fetchNextPage();
  }, [inView, hasNextPage, isFetchingNextPage, fetchNextPage]);

  if (isLoading) return <Spinner />;
  if (error) return <ErrorMessage />;

  const items = data?.pages.flatMap(p => p.items) ?? [];
  if (items.length === 0) return <EmptyState />;

  return (
    <ul>
      {items.map(item => <PostItem key={item.id} post={item} />)}
      {hasNextPage && (
        <li ref={ref}>
          {isFetchingNextPage ? <Spinner /> : null}
        </li>
      )}
    </ul>
  );
}
```

## 5. 가상화 (수천 row)

→ list 가 매우 크면 [[../performance/virtual-list]] 의 가상화 결합:

```tsx
import InfiniteLoader from 'react-window-infinite-loader';
import { FixedSizeList } from 'react-window';

const items = data?.pages.flatMap(p => p.items) ?? [];
const itemCount = hasNextPage ? items.length + 1 : items.length;
const isLoaded = (i: number) => !hasNextPage || i < items.length;

<InfiniteLoader
  isItemLoaded={isLoaded}
  itemCount={itemCount}
  loadMoreItems={() => !isFetchingNextPage && fetchNextPage()}
>
  {({ onItemsRendered, ref }) => (
    <FixedSizeList
      ref={ref}
      onItemsRendered={onItemsRendered}
      height={600}
      itemCount={itemCount}
      itemSize={80}
      width="100%"
    >
      {({ index, style }) => (
        <div style={style}>
          {isLoaded(index) ? items[index].title : '로딩...'}
        </div>
      )}
    </FixedSizeList>
  )}
</InfiniteLoader>
```

## 6. 스크롤 위치 복원 — 뒤로가기 시

```tsx
// 1. 스크롤 위치 저장
useEffect(() => {
  return () => sessionStorage.setItem('post-scroll', String(window.scrollY));
}, []);

// 2. 마운트 시 복원
useEffect(() => {
  const y = Number(sessionStorage.getItem('post-scroll') ?? 0);
  window.scrollTo(0, y);
}, []);
```

→ 무한 스크롤 + 상세 페이지 갔다 돌아올 때 스크롤 위치 + 데이터 유지. react-query 의 캐시 + 스크롤 복원 = 완벽.

## 7. 빈 페이지 / 마지막 페이지 처리

```ts
getNextPageParam: (lastPage) => {
  // 더 없으면 undefined return → hasNextPage = false
  if (lastPage.items.length < 20) return undefined;
  return lastPage.nextPage;
}
```

서버 응답 케이스에 따라:
- `nextPage: null` 명시.
- `items.length < pageSize` 로 끝 판단.
- `total` 와 비교.

## 8. cursor 기반 paging

```ts
queryFn: ({ pageParam }) => api.get(`/posts?cursor=${pageParam}`).then(r => r.data),
initialPageParam: '',
getNextPageParam: (last) => last.nextCursor,
// { items, nextCursor: 'eyJhZG...' | null }
```

→ offset 기반보다 안정적 (insert 중에도 중복/누락 적음).

## 9. mutation 후 cache 갱신

```ts
// 새 post 추가
const qc = useQueryClient();

const { mutate: createPost } = useMutation({
  mutationFn: api.createPost,
  onSuccess: () => qc.invalidateQueries({ queryKey: ['posts'] }),
});
```

→ invalidate 면 모든 page 다시 fetch. 단순.

더 정교: optimistic update + 첫 페이지에만 추가.

## 10. 사용자 트리거 vs 자동

```tsx
// 자동 (스크롤 끝 도달 시)
{hasNextPage && <div ref={ref}>...</div>}

// 사용자 클릭 ("더 보기" 버튼)
{hasNextPage && (
  <button onClick={() => fetchNextPage()} disabled={isFetchingNextPage}>
    더 보기
  </button>
)}

// 혼합 — 5 페이지마다 자동, 그 후 버튼
const autoLoad = data.pages.length < 5;
```

→ 무한 자동은 사용자가 footer 못 봄. UX 따라.

## 11. answer-fe / job-answer-fe 패턴

```tsx
// hooks/usePostsInfinite.ts
export const usePostsInfinite = () => {
  return useInfiniteQuery({
    queryKey: ['posts'],
    queryFn: ({ pageParam = 0 }) => postApi.list({ page: pageParam }),
    initialPageParam: 0,
    getNextPageParam: (last, all) => last.hasNext ? all.length : undefined,
  });
};

// PostListPage.tsx
const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = usePostsInfinite();
const { ref, inView } = useInView();

useEffect(() => {
  if (inView && hasNextPage) fetchNextPage();
}, [inView, hasNextPage]);

const posts = data?.pages.flatMap(p => p.items) ?? [];

return (
  <>
    {posts.map(p => <PostCard key={p.id} post={p} />)}
    <div ref={ref}>{isFetchingNextPage && <Spinner />}</div>
  </>
);
```

## 12. 함정

1. **trigger 안 보이는 위치** — `<div ref={ref} />` 가 화면 밖이면 fire 안 됨. list 끝에 배치.
2. **`!hasNextPage` 확인 안 함** — 끝에 도달 후 무한 fetch 시도.
3. **`isFetchingNextPage` 확인 안 함** — 동시 N 번 fetch.
4. **fetchNextPage 의존성** — react-query 가 매번 새 ref. useEffect deps 에 넣을 때 stable.
5. **mutation 후 invalidate** — 잊으면 화면 갱신 안 됨.
6. **스크롤 위치 안 복원** — 뒤로가기 시 처음부터.
7. **cursor 의 인코딩** — base64 / URL-safe.

## 13. 외부 자료

- [TanStack Query — useInfiniteQuery](https://tanstack.com/query/latest/docs/framework/react/reference/useInfiniteQuery)
- [react-intersection-observer](https://www.npmjs.com/package/react-intersection-observer)
- [MDN IntersectionObserver](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API)

## 14. 관련

- [[recipes]]
- [[../server-state/react-query]]
- [[../performance/virtual-list]]
