---
title: "Reanimated + Gesture Handler — 60fps 애니메이션"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, reanimated, gesture-handler, animation]
---

# Reanimated + Gesture Handler

**[[performance|↑ Performance Hub]]**

> 60fps 부드러운 애니메이션 + 제스처. **JS thread 안 거치고 native thread 에서 실행**.

## 1. 왜 reanimated?

RN 의 기본 `Animated` API:
- 일부 동작 JS bridge 거침.
- 무거운 JS 로직 시 frame drop.

`react-native-reanimated`:
- **모든 애니메이션 = native thread 에서**.
- JS thread bottle neck 무관.
- gesture / scroll 과 synced.

## 2. 설치

```bash
yarn add react-native-reanimated react-native-gesture-handler
cd ios && pod install
```

`babel.config.js`:
```js
module.exports = {
  presets: ['module:@react-native/babel-preset'],
  plugins: [
    // ... 다른 plugin
    'react-native-reanimated/plugin',          // 마지막에 (필수)
  ],
};
```

`App.tsx`:
```tsx
import 'react-native-gesture-handler';            // index.js 의 맨 위
import { GestureHandlerRootView } from 'react-native-gesture-handler';

<GestureHandlerRootView style={{ flex: 1 }}>
  <App />
</GestureHandlerRootView>
```

## 3. 기본 — useSharedValue + useAnimatedStyle

```tsx
import Animated, { useSharedValue, useAnimatedStyle, withSpring, withTiming } from 'react-native-reanimated';

function Box() {
  const offset = useSharedValue(0);

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ translateX: offset.value }],
  }));

  return (
    <>
      <Animated.View style={[styles.box, animatedStyle]} />
      <Pressable onPress={() => { offset.value = withSpring(100); }}>
        <Text>오른쪽으로</Text>
      </Pressable>
    </>
  );
}
```

- `useSharedValue` — UI thread 의 값.
- `useAnimatedStyle` — UI thread 의 style 계산.
- `withSpring` / `withTiming` / `withDecay` — animation function.

## 4. withSpring / withTiming

```ts
offset.value = withTiming(100, { duration: 500 });
offset.value = withSpring(100, { damping: 10, stiffness: 100 });
offset.value = withDecay({ velocity: 200, clamp: [0, 300] });

// chain
offset.value = withSequence(
  withTiming(100),
  withDelay(500, withTiming(0)),
);

// repeat
offset.value = withRepeat(withTiming(100), -1, true);   // 무한, 왕복
```

## 5. interpolate — 입력 ↔ 출력 매핑

```ts
import { interpolate, Extrapolate } from 'react-native-reanimated';

const animatedStyle = useAnimatedStyle(() => ({
  opacity: interpolate(
    offset.value,
    [0, 100, 200],            // 입력
    [0, 1, 0],                // 출력
    Extrapolate.CLAMP,
  ),
}));
```

## 6. Gesture Handler — Pan / Pinch / Swipe

```tsx
import { GestureDetector, Gesture } from 'react-native-gesture-handler';
import Animated, { useSharedValue, useAnimatedStyle } from 'react-native-reanimated';

function Draggable() {
  const translateX = useSharedValue(0);
  const translateY = useSharedValue(0);

  const pan = Gesture.Pan()
    .onUpdate((e) => {
      translateX.value = e.translationX;
      translateY.value = e.translationY;
    })
    .onEnd(() => {
      translateX.value = withSpring(0);
      translateY.value = withSpring(0);
    });

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [
      { translateX: translateX.value },
      { translateY: translateY.value },
    ],
  }));

  return (
    <GestureDetector gesture={pan}>
      <Animated.View style={[styles.box, animatedStyle]} />
    </GestureDetector>
  );
}
```

### 다른 gesture
- `Gesture.Tap()` — 탭.
- `Gesture.LongPress()` — 길게 누름.
- `Gesture.Pinch()` — 핀치 줌.
- `Gesture.Rotation()` — 회전.
- `Gesture.Fling()` — 빠른 swipe.

### 조합
```ts
const composed = Gesture.Simultaneous(pan, pinch);
const exclusive = Gesture.Exclusive(tap, doubleTap);
```

## 7. layout animation (자동)

```ts
import Animated, { Layout, FadeIn, FadeOut } from 'react-native-reanimated';

<Animated.View
  entering={FadeIn}                       // 마운트 시
  exiting={FadeOut}                       // 언마운트 시
  layout={Layout.springify()}              // 자식 / 위치 변화 시 자동
>
  ...
</Animated.View>
```

→ list 의 item 추가 / 삭제 시 부드러운 layout 전환. 별도 코드 없이.

## 8. worklet — UI thread 함수

```ts
function expensiveCalc(input: number) {
  'worklet';
  return input * 2;
}

const style = useAnimatedStyle(() => {
  const out = expensiveCalc(offset.value);
  return { transform: [{ translateX: out }] };
});
```

→ `'worklet'` 문자열 = UI thread 에서 실행. JS 의 일부 API 만 사용 가능 (Math, basic 함수). `console.log` X (UI thread 라서).

## 9. JS thread 와 통신 — runOnJS

```ts
import { runOnJS } from 'react-native-reanimated';

const pan = Gesture.Pan()
  .onEnd(() => {
    runOnJS(navigation.goBack)();    // JS thread 호출
  });
```

→ worklet 에서 JS 호출. setState, navigation 등.

## 10. job-answer-app-rn 의 패턴

### 녹음 버튼 (펄스 애니메이션)
```tsx
const scale = useSharedValue(1);

useEffect(() => {
  if (isRecording) {
    scale.value = withRepeat(withTiming(1.2, { duration: 600 }), -1, true);
  } else {
    scale.value = withTiming(1);
  }
}, [isRecording]);

const animatedStyle = useAnimatedStyle(() => ({
  transform: [{ scale: scale.value }],
}));

<Animated.View style={[styles.button, animatedStyle]}>
  <Icon name="mic" size={32} />
</Animated.View>
```

→ Lottie 대신 직접 reanimated.

### Bottom Sheet swipe
```tsx
const translateY = useSharedValue(0);

const pan = Gesture.Pan()
  .onUpdate((e) => {
    translateY.value = Math.max(0, e.translationY);
  })
  .onEnd((e) => {
    if (e.translationY > 100) {
      translateY.value = withTiming(SCREEN_HEIGHT, {}, () => runOnJS(onClose)());
    } else {
      translateY.value = withSpring(0);
    }
  });
```

## 11. Animated (built-in) vs reanimated

| | Animated (RN built-in) | reanimated |
| --- | --- | --- |
| thread | 일부 native | **완전 native** |
| 성능 | 보통 | 빠름 |
| API | 명령형 | 선언형 |
| 학습 | 쉬움 | 약간 더 |

→ 신규 = reanimated.

## 12. 함정

1. **babel plugin 누락** — animation 안 됨. babel.config.js 의 마지막에.
2. **`'worklet'` 누락** — runOnUI 함수 호출 시 에러.
3. **`useSharedValue` 의 React state 와 혼동** — re-render 안 함. UI 만.
4. **`onUpdate` 안 React state 호출** — runOnJS 필요.
5. **`GestureHandlerRootView` 누락** — Gesture 동작 X.
6. **layout animation + Flat list** — 잘 안 됨. itemLayoutAnimation 별도.
7. **dev mode 의 reload** — shared value 가 초기값으로 reset.

## 13. 다른 옵션

- **moti** — reanimated 위 declarative wrapper. 더 쉬움.
- **framer-motion-rn** — web framer-motion 의 RN port (미성숙).
- **lottie-react-native** — 디자이너 제작 JSON. [[../ui/lottie]].

## 14. 외부 자료

- [react-native-reanimated 공식](https://docs.swmansion.com/react-native-reanimated/)
- [react-native-gesture-handler 공식](https://docs.swmansion.com/react-native-gesture-handler/)
- [Reanimated YouTube tutorials](https://www.youtube.com/@WilliamCandillon)

## 15. 관련

- [[performance]]
- [[../ui/lottie]]
- [[flatlist]]
