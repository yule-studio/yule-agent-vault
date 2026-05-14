---
title: "FlatList — 가상화 list 의 정석"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, flatlist, virtualization, performance]
---

# FlatList

**[[performance|↑ Performance Hub]]**

> RN 의 큰 list 의 표준. 보이는 row 만 렌더 = 가상화.

## 1. ScrollView vs FlatList

```tsx
// ScrollView — 모든 row 즉시 DOM
<ScrollView>
  {items.map(item => <Row key={item.id} item={item} />)}
</ScrollView>
```

→ 100+ row 면 첫 render 매우 느림 + 스크롤 끊김.

```tsx
// FlatList — 보이는 row만 + 위아래 buffer
<FlatList
  data={items}
  keyExtractor={item => String(item.id)}
  renderItem={({ item }) => <Row item={item} />}
/>
```

→ 1000+ row 도 부드러움.

## 2. 기본 prop

```tsx
<FlatList
  data={items}                                  // 배열
  keyExtractor={item => String(item.id)}        // unique key
  renderItem={({ item, index }) => <Row item={item} />}
  
  // 헤더 / 푸터 / 빈 상태
  ListHeaderComponent={<Header />}
  ListFooterComponent={isLoading ? <Spinner /> : null}
  ListEmptyComponent={<EmptyState />}
  
  // 구분선
  ItemSeparatorComponent={() => <Separator />}
  
  // 무한 스크롤
  onEndReached={loadMore}
  onEndReachedThreshold={0.5}                  // 0~1, 비율
  
  // pull to refresh
  refreshing={refreshing}
  onRefresh={onRefresh}
/>
```

## 3. 성능 옵션

```tsx
<FlatList
  // 첫 render
  initialNumToRender={10}        // 첫 10개만 (보이는 영역 만큼)
  
  // 스크롤 시
  maxToRenderPerBatch={10}       // batch 당 10 개씩
  windowSize={5}                 // viewport 의 5 배만큼 buffer (위 2.5 + 아래 2.5)
  updateCellsBatchingPeriod={50} // batch 사이 ms
  
  // 메모리
  removeClippedSubviews          // 화면 밖 row 의 view 제거 (Android 효과 큼)
  
  // 정확한 layout
  getItemLayout={(_, index) => ({ length: 80, offset: 80 * index, index })}
  // ↑ 모든 row 같은 height 일 때
/>
```

### `getItemLayout` 이 강력한 이유
- FlatList 가 row 의 height 를 측정하지 않고 알 수 있음.
- 스크롤 위치 즉시 계산 가능.
- "100번째 row 로 가기" 도 즉시.

→ 모든 row 같은 height (예: 80) 면 사용 강력 추천.

## 4. row 의 메모이제이션

```tsx
// ❌ inline component — 매 render 새 type
<FlatList
  renderItem={({ item }) => (
    <View>
      <Text>{item.name}</Text>
    </View>
  )}
/>

// ✅ 외부 component + React.memo
const Row = React.memo(({ item }: { item: Item }) => (
  <View style={styles.row}>
    <Text>{item.name}</Text>
  </View>
));

const renderItem = useCallback(({ item }) => <Row item={item} />, []);

<FlatList renderItem={renderItem} />
```

→ row 의 props 가 같으면 re-render skip.

## 5. row 의 callback prop — useCallback

```tsx
const onPress = useCallback((id: number) => {
  navigation.navigate('Detail', { id });
}, [navigation]);

const renderItem = useCallback(({ item }) => (
  <Row item={item} onPress={onPress} />
), [onPress]);

const Row = React.memo(({ item, onPress }) => (
  <Pressable onPress={() => onPress(item.id)}>
    <Text>{item.name}</Text>
  </Pressable>
));
```

→ onPress 가 stable → Row 의 props 가 stable → memo 효과.

## 6. keyExtractor 의 중요성

```tsx
// ❌ index 사용 — 삭제/삽입 시 잘못된 row 가 re-render
keyExtractor={(_, i) => String(i)}

// ✅ 안정적 id
keyExtractor={item => String(item.id)}
```

→ web 의 `key={i}` 와 동일한 함정.

## 7. SectionList — 섹션 구분

```tsx
<SectionList
  sections={[
    { title: '오늘', data: todayItems },
    { title: '어제', data: yesterdayItems },
  ]}
  keyExtractor={(item, i) => item.id + i}
  renderItem={({ item }) => <Row item={item} />}
  renderSectionHeader={({ section }) => <Header title={section.title} />}
/>
```

## 8. VirtualizedList — base class

→ FlatList / SectionList 의 부모. 직접 사용 드뭄.

## 9. FlashList — 빠른 대안

```bash
yarn add @shopify/flash-list
```

```tsx
import { FlashList } from '@shopify/flash-list';

<FlashList
  data={items}
  renderItem={({ item }) => <Row item={item} />}
  estimatedItemSize={80}     // 평균 height
/>
```

→ Shopify 가 만든 FlatList replacement. **수 배 빠름** (특히 큰 list).

→ FlatList 와 거의 호환 API.

## 10. 무한 스크롤 — react-query + FlatList

```tsx
const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useInfiniteQuery({
  queryKey: ['posts'],
  queryFn: ({ pageParam = 0 }) => api.get(`/posts?page=${pageParam}`).then(r => r.data),
  initialPageParam: 0,
  getNextPageParam: (last) => last.nextPage,
});

const items = data?.pages.flatMap(p => p.items) ?? [];

<FlatList
  data={items}
  keyExtractor={item => String(item.id)}
  renderItem={renderItem}
  onEndReached={() => hasNextPage && !isFetchingNextPage && fetchNextPage()}
  onEndReachedThreshold={0.5}
  ListFooterComponent={isFetchingNextPage ? <Spinner /> : null}
/>
```

자세히 [[../../../frontend/react/recipes/infinite-scroll|web infinite-scroll]] (RN 도 동일 사상).

## 11. Pull to refresh

```tsx
const [refreshing, setRefreshing] = useState(false);

const onRefresh = async () => {
  setRefreshing(true);
  await queryClient.invalidateQueries({ queryKey: ['posts'] });
  setRefreshing(false);
};

<FlatList refreshing={refreshing} onRefresh={onRefresh} ... />
```

## 12. 스크롤 컨트롤

```tsx
const ref = useRef<FlatList<Item>>(null);

// 처음으로
ref.current?.scrollToOffset({ offset: 0, animated: true });

// 특정 index 로
ref.current?.scrollToIndex({ index: 50, animated: true });

// 특정 item 으로 (getItemLayout 또는 onScrollToIndexFailed 처리)
ref.current?.scrollToItem({ item: targetItem });

<FlatList
  ref={ref}
  onScrollToIndexFailed={(info) => {
    setTimeout(() => ref.current?.scrollToIndex({ index: info.index }), 100);
  }}
/>
```

## 13. 가로 list (horizontal)

```tsx
<FlatList horizontal data={...} renderItem={...} showsHorizontalScrollIndicator={false} />
```

### Carousel — 한 row 가 화면 가득
```tsx
<FlatList
  horizontal
  pagingEnabled                  // snap to page
  showsHorizontalScrollIndicator={false}
  data={slides}
  renderItem={({ item }) => (
    <View style={{ width: screenWidth }}>...</View>
  )}
/>
```

## 14. job-answer-app-rn 패턴

```tsx
// QuestionList
import { useInfiniteQuery } from '@tanstack/react-query';

function QuestionList() {
  const { data, fetchNextPage, hasNextPage } = useQuestionsInfinite();
  const questions = data?.pages.flatMap(p => p.items) ?? [];

  const renderItem = useCallback(({ item }) => (
    <QuestionItem question={item} onPress={navigateToDetail} />
  ), [navigateToDetail]);

  return (
    <FlatList
      data={questions}
      keyExtractor={q => String(q.id)}
      renderItem={renderItem}
      ItemSeparatorComponent={() => <View style={{ height: 1, backgroundColor: '#eee' }} />}
      onEndReached={() => hasNextPage && fetchNextPage()}
      onEndReachedThreshold={0.5}
      ListEmptyComponent={<EmptyState />}
    />
  );
}

const QuestionItem = React.memo(({ question, onPress }) => (
  <Pressable onPress={() => onPress(question.id)}>
    <Text>{question.title}</Text>
  </Pressable>
));
```

## 15. 함정

1. **inline renderItem** — 매 render 새 함수. useCallback.
2. **index 를 key** — reorder / insert / delete 시 state 엉킴.
3. **row 의 무거운 inline 스타일** — useMemo 또는 StyleSheet.
4. **getItemLayout 의 잘못된 length** — 스크롤 위치 어긋남.
5. **`removeClippedSubviews` 의 부작용** — 일부 모달 / animated view 의 시각 깨짐.
6. **무한 onEndReached** — `!isFetchingNextPage` 가드 누락.
7. **ScrollView 안 FlatList** — 가상화 동작 X. nested 면 FlatList 가 main.

## 16. 외부 자료

- [reactnative.dev — FlatList](https://reactnative.dev/docs/flatlist)
- [reactnative.dev — Optimizing FlatList](https://reactnative.dev/docs/optimizing-flatlist-configuration)
- [FlashList by Shopify](https://shopify.github.io/flash-list/)

## 17. 관련

- [[performance]]
- [[../core-concepts/rn-components]]
- [[../../../frontend/react/performance/virtual-list|web virtual-list]]
