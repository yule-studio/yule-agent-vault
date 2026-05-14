---
title: "Virtual List — 큰 list 의 성능 (react-window)"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, performance, virtual-list, react-window, react-virtual]
---

# Virtual List

**[[performance|↑ Performance Hub]]**

> 수천 row 의 list. **화면에 보이는 row 만 렌더**.

## 1. 왜 필요한가?

```tsx
{thousand_items.map(it => <Row item={it} />)}
```

→ 1000 row 의 DOM 동시 생성:
- 첫 render 매우 느림.
- 스크롤 시 layout / paint 부담.
- 메모리 폭증.

## 2. 가상화의 원리

```
화면 (viewport)
├──────────────┤
│ [Row 5]      │
│ [Row 6]      │  ← 보이는 영역만 실제 DOM
│ [Row 7]      │
├──────────────┤
                   Row 0~4, 8~999 는 DOM 없음 (height 만 차지)
```

→ 보이는 row + 약간의 buffer 만 실제 DOM.

## 3. react-window — 사실상 표준

```bash
yarn add react-window
yarn add -D @types/react-window
```

### Fixed height

```tsx
import { FixedSizeList } from 'react-window';

function MyList({ items }: { items: Item[] }) {
  return (
    <FixedSizeList
      height={400}              // 컨테이너 height
      itemCount={items.length}
      itemSize={48}             // 각 row height
      width="100%"
    >
      {({ index, style }) => (
        <div style={style}>
          {items[index].name}
        </div>
      )}
    </FixedSizeList>
  );
}
```

→ **`style` 을 반드시 row 의 outer div 에 전달**. 위치 + height 설정.

### Variable height — VariableSizeList

```tsx
import { VariableSizeList } from 'react-window';
import { useRef } from 'react';

function MyList({ items }) {
  const ref = useRef<VariableSizeList>(null);

  const getSize = (index: number) => {
    // 각 item 의 height 반환
    return items[index].text.length > 100 ? 80 : 48;
  };

  return (
    <VariableSizeList
      ref={ref}
      height={400}
      itemCount={items.length}
      itemSize={getSize}
      width="100%"
    >
      {({ index, style }) => (
        <div style={style}>{items[index].text}</div>
      )}
    </VariableSizeList>
  );
}
```

→ height 동적 측정 필요 시 더 복잡. react-virtual / virtuoso 가 더 강력.

### Grid (2D)

```tsx
import { FixedSizeGrid } from 'react-window';

<FixedSizeGrid
  columnCount={5}
  columnWidth={100}
  rowCount={1000}
  rowHeight={50}
  height={400}
  width={600}
>
  {({ columnIndex, rowIndex, style }) => (
    <div style={style}>{rowIndex}, {columnIndex}</div>
  )}
</FixedSizeGrid>
```

## 4. 무한 스크롤 + 가상화 — `react-window-infinite-loader`

```bash
yarn add react-window-infinite-loader
```

```tsx
import InfiniteLoader from 'react-window-infinite-loader';
import { FixedSizeList } from 'react-window';

function InfiniteList({ items, loadMore, hasMore }) {
  const itemCount = hasMore ? items.length + 1 : items.length;
  const isLoaded = (index: number) => !hasMore || index < items.length;

  return (
    <InfiniteLoader
      isItemLoaded={isLoaded}
      itemCount={itemCount}
      loadMoreItems={loadMore}
    >
      {({ onItemsRendered, ref }) => (
        <FixedSizeList
          ref={ref}
          onItemsRendered={onItemsRendered}
          height={500}
          itemCount={itemCount}
          itemSize={48}
          width="100%"
        >
          {({ index, style }) => (
            <div style={style}>
              {!isLoaded(index) ? '로딩...' : items[index].name}
            </div>
          )}
        </FixedSizeList>
      )}
    </InfiniteLoader>
  );
}
```

## 5. react-virtual (TanStack) — modern 대안

```bash
yarn add @tanstack/react-virtual
```

```tsx
import { useVirtualizer } from '@tanstack/react-virtual';
import { useRef } from 'react';

function MyList({ items }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const rowVirtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 48,
  });

  return (
    <div ref={parentRef} style={{ height: 400, overflow: 'auto' }}>
      <div style={{ height: rowVirtualizer.getTotalSize(), position: 'relative' }}>
        {rowVirtualizer.getVirtualItems().map(vi => (
          <div
            key={vi.index}
            style={{
              position: 'absolute',
              top: 0, left: 0, width: '100%',
              height: vi.size,
              transform: `translateY(${vi.start}px)`,
            }}
          >
            {items[vi.index].name}
          </div>
        ))}
      </div>
    </div>
  );
}
```

→ react-window 의 후속. dynamic height 측정 자동, grid / row / column 통일 API.

## 6. virtuoso — 더 high-level

```bash
yarn add react-virtuoso
```

```tsx
import { Virtuoso } from 'react-virtuoso';

<Virtuoso
  style={{ height: 400 }}
  data={items}
  itemContent={(index, item) => <div>{item.name}</div>}
  endReached={() => loadMore()}
/>
```

→ 가장 쉬움. dynamic height 자동. infinite 도 한 줄.

## 7. 언제 가상화?

- list 가 100 ~ 200 개 넘어가면 검토.
- 첫 render 측정해 100ms 넘으면 적용.
- 단순 list (작은 row, 100 개 안 됨) = 가상화 불필요.

## 8. 가상화의 단점 / 주의

1. **Ctrl+F 검색 안 됨** — DOM 에 없는 항목 검색 X. (Chrome 의 일부 버전은 됨)
2. **a11y** — screen reader 가 보이지 않는 항목 안 읽음.
3. **scroll-to** — 특정 item 으로 스크롤은 lib 의 method 사용.
4. **row 안 input focus** — scroll 해서 DOM 에서 사라지면 focus 잃음.
5. **dynamic height** — 측정 비용. VariableSizeList 또는 virtuoso.

## 9. answer-fe 의 사용 예

```ts
// package.json
"react-window": "^1.8.x",
```

```tsx
// 질문 list 의 큰 결과
<FixedSizeList
  height={600}
  itemCount={questions.length}
  itemSize={120}
  width="100%"
>
  {({ index, style }) => (
    <div style={style}>
      <QuestionItem question={questions[index]} />
    </div>
  )}
</FixedSizeList>
```

→ infinite scroll + 가상화 결합.

## 10. 함정

1. **`style` 누락** — row 가 한 위치에 겹침.
2. **row 의 box-sizing** — `style` 의 height 와 실제 content height 다름.
3. **memo 없는 row** — 스크롤 시 매번 re-render. `React.memo` 권장.
4. **fixed height 가정** — 실제 dynamic 인데 fixed 쓰면 잘림.
5. **scrollTo 누락** — modal 열고 닫으면 스크롤 위치 잃음. ref + scrollToItem.
6. **TanStack Query 와 결합** — infinite query 의 page 평탄화 필요:
   ```ts
   const items = data?.pages.flatMap(p => p.items) ?? [];
   ```

## 11. 외부 자료

- [react-window](https://react-window.vercel.app/)
- [@tanstack/react-virtual](https://tanstack.com/virtual/latest)
- [react-virtuoso](https://virtuoso.dev/)

## 12. 관련

- [[performance]]
- [[../recipes/infinite-scroll]]
- [[../server-state/react-query]]
