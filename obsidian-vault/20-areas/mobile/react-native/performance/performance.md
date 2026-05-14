---
title: "Performance in RN — 60fps 의 길"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, performance, fps, hub]
---

# Performance in RN

**[[../react-native|↑ RN Hub]]**

> "느려진 후" 최적화. **60fps 유지** + **bundle / 메모리 최적화**.

## 1. 측정 도구

- **Dev Menu → Show Perf Monitor** — FPS / RAM 실시간.
- **React DevTools Profiler** — 어느 컴포넌트가 느린지.
- **Flipper / Hermes inspector** — heap / network / log.
- **Xcode Instruments** — iOS native 분석.
- **Android Studio Profiler** — Android native + JS.

## 2. 영역

| | 노트 |
| --- | --- |
| **List 가상화** | [[flatlist]] |
| **메모이제이션** | [[../../../frontend/react/performance/memoization|memoization]] (web 동일) |
| **애니메이션** | [[reanimated-gesture]] |
| **이미지 최적화** | (아래 §5) |
| **번들 / Hermes** | (아래 §6) |
| **Re-render** | [[../../../frontend/react/pitfalls/re-render-loops|re-render-loops]] |

## 3. 60fps 유지의 핵심 — JS thread

RN 의 thread 모델:
- **JS thread** — React 로직 (state, render).
- **UI thread (main)** — 실제 화면 그리기.
- **bridge** — 두 thread 의 통신 (느림).

→ JS thread 가 무거우면 → bridge 가 막힘 → UI 가 frame drop.

대책:
- 무거운 계산을 `useMemo` / web worker (RN 에선 별도 lib).
- **`react-native-reanimated`** — 애니메이션을 UI thread 에 직접.
- list 는 FlatList 가상화.

## 4. FlatList 최적화 — [[flatlist]] 자세히

```tsx
<FlatList
  data={items}
  renderItem={renderItem}
  keyExtractor={item => String(item.id)}
  
  // 핵심 옵션
  initialNumToRender={10}            // 첫 render 시 만들 row 수
  maxToRenderPerBatch={10}           // 매 batch 의 row
  windowSize={5}                      // 보이는 영역의 viewport 배수
  removeClippedSubviews             // 화면 밖 row 제거
  
  // row 자체 — React.memo
  renderItem={({ item }) => <MemoRow item={item} />}
/>

const MemoRow = React.memo(({ item }) => <Row item={item} />);
```

## 5. 이미지 최적화

### 작은 이미지로
```tsx
// 매 row 의 큰 원본 (1080x1080) → 100x100 thumbnail 별도
<Image source={{ uri: item.thumbnail }} style={{ width: 100, height: 100 }} />
```

### FastImage — 캐싱
```bash
yarn add react-native-fast-image
cd ios && pod install
```

```tsx
import FastImage from 'react-native-fast-image';

<FastImage
  source={{ uri, priority: FastImage.priority.normal, cache: FastImage.cacheControl.immutable }}
  style={{ width: 100, height: 100 }}
  resizeMode={FastImage.resizeMode.cover}
/>
```

→ 메모리 / 디스크 캐싱 + 진보된 priority.

### 이미지 prefetch
```ts
Image.prefetch('https://...')
```

## 6. Hermes — JS 엔진

RN 0.70+ 의 기본 JS 엔진. JSC (옛 default) 보다:
- 시작 시간 단축.
- 메모리 감소.
- bytecode pre-compile.

→ 켜져 있는지 확인:
- `metro.config.js` 의 `resolver.unstable_enableSymlinks: true`.
- `android/app/build.gradle` 의 `enableHermes: true`.
- `ios/Podfile` 의 `:hermes_enabled => true`.

→ 기본 새 RN 프로젝트는 자동 켜짐.

## 7. console.log — production 제거

```js
// babel.config.js
plugins: [
  // ... 다른 plugin
  ['transform-remove-console', { exclude: ['error', 'warn'] }],
],
```

```bash
yarn add -D babel-plugin-transform-remove-console
```

→ production bundle 에서 console.log 제거. 성능 + 보안.

## 8. inline 함수 / 객체 — memo 와 함께

→ web 과 동일. [[../../../frontend/react/performance/memoization|memoization]].

```tsx
// ❌ 매 render 새 객체
<Child style={{ padding: 16 }} />

// ✅ 외부 또는 useMemo
const style = StyleSheet.create({ child: { padding: 16 } });
<Child style={style.child} />
```

## 9. Bridge 최소화 — Native Module / TurboModule

- 새 아키텍처 (Fabric + TurboModules) — bridge 제거.
- 큰 app 의 perf 향상.
- 신규 프로젝트 = 자동 활성화 (0.74+).

## 10. background 작업

```ts
import { AppState } from 'react-native';

useEffect(() => {
  const sub = AppState.addEventListener('change', (status) => {
    if (status === 'background') {
      // 무거운 작업 중단 (interval, websocket)
    }
  });
  return () => sub.remove();
}, []);
```

→ background 에서 리소스 절약.

## 11. memory leak 패턴

| 누수 | 해결 |
| --- | --- |
| setInterval / setTimeout | useEffect cleanup |
| addEventListener | removeEventListener |
| WebSocket / Socket.IO | deactivate / disconnect |
| Voice / Camera | Voice.destroy(), camera release |
| 큰 객체의 ref 보관 | unmount 시 ref.current = null |

## 12. 학습 우선순위

1. **[[flatlist]]** — 가장 흔한 성능 문제.
2. **memoization** — [[../../../frontend/react/performance/memoization|web 의 memoization]] 그대로.
3. **이미지 (FastImage)**.
4. **[[reanimated-gesture]]** — 부드러운 애니메이션.
5. **production console 제거**.
6. **번들 분석** — `npx react-native-bundle-visualizer`.

## 13. 함정

1. **모든 곳 React.memo** — 비용만 늘림. 측정 후.
2. **FlatList 의 inline renderItem** — 매 render 새 함수.
3. **이미지 size 명시 안 함** — layout shift.
4. **새 객체 / 함수 prop** — memo 무력화.
5. **무거운 useEffect** — JS thread block.
6. **너무 많은 native module** — bundle / start 시간.

## 14. 다음 단계

- [[flatlist]] — 가상화 list
- [[reanimated-gesture]] — 60fps 애니메이션

## 15. 외부 자료

- [reactnative.dev — Performance](https://reactnative.dev/docs/performance)
- [Hermes](https://hermesengine.dev/)
- [React Native Reanimated](https://docs.swmansion.com/react-native-reanimated/)

## 16. 관련

- [[../core-concepts/rn-components]] — FlatList
- [[../../../frontend/react/performance/memoization|web memoization]]
- [[../pitfalls/pitfalls]]
