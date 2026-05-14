---
title: "SafeArea + StatusBar — 노치 / 다이내믹 아일랜드 / 상태바"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, safe-area, status-bar, notch]
---

# SafeArea + StatusBar

**[[core-concepts|↑ Core Concepts]]**

> iPhone 의 노치 / Dynamic Island, Android 의 punch hole + 상태바를 침범하지 않게.

## 1. 문제 — 노치를 피해야

```
┌──────────────┐  ← 시계 / 배터리 영역 (status bar)
│ ⚪ 노치 ⚪    │
├──────────────┤
│              │
│  여기에 내용 │
│              │
├──────────────┤
│ 홈 인디케이터 │  ← iOS 하단 홈 바
└──────────────┘
```

→ 노치 / 시계 / 홈바 영역 = **unsafe area**. 그 안쪽 = **safe area**.

## 2. SafeAreaView — 가장 단순

```tsx
import { SafeAreaView } from 'react-native-safe-area-context';

<SafeAreaView style={{ flex: 1 }}>
  <Text>안전한 영역</Text>
</SafeAreaView>
```

→ 자동으로 노치 / 상태바 / 홈바 영역 패딩.

### 두 종류의 SafeAreaView

```tsx
// ❌ react-native (기본) — iOS 만 동작, 구식
import { SafeAreaView } from 'react-native';

// ✅ react-native-safe-area-context — 권장
import { SafeAreaView } from 'react-native-safe-area-context';
```

### 설치
```bash
yarn add react-native-safe-area-context
cd ios && pod install
```

### SafeAreaProvider — root 필수
```tsx
import { SafeAreaProvider } from 'react-native-safe-area-context';

<SafeAreaProvider>
  <App />
</SafeAreaProvider>
```

## 3. edges prop — 일부 영역만

```tsx
<SafeAreaView edges={['top']} style={{ flex: 1 }}>
  {/* 위만 안전, 아래는 자유 (예: 하단 탭이 있을 때) */}
</SafeAreaView>

<SafeAreaView edges={['top', 'bottom']}>
  {/* 위 + 아래 */}
</SafeAreaView>

<SafeAreaView edges={['top', 'left', 'right', 'bottom']}>
  {/* 모두 (기본) */}
</SafeAreaView>
```

→ 하단 탭이 있는 화면은 `['top']` 만.

## 4. useSafeAreaInsets — 실제 픽셀 값 접근

```tsx
import { useSafeAreaInsets } from 'react-native-safe-area-context';

function Header() {
  const insets = useSafeAreaInsets();
  return (
    <View style={{ paddingTop: insets.top, paddingBottom: 0, backgroundColor: 'white' }}>
      <Text>커스텀 header</Text>
    </View>
  );
}
```

→ `insets.top` = 노치 / 상태바 높이.
→ `insets.bottom` = 홈 인디케이터 높이.
→ `insets.left`, `insets.right` = 가로 노치 (드물게).

## 5. StatusBar — 시계 영역 스타일

```tsx
import { StatusBar } from 'react-native';

<StatusBar
  barStyle="dark-content"          // 'default' | 'light-content' | 'dark-content'
  backgroundColor="white"          // Android 만
  translucent={false}              // Android: 상태바 투명?
  hidden={false}
/>
```

### barStyle
- `light-content` — 흰 텍스트 (어두운 배경 위).
- `dark-content` — 검은 텍스트 (밝은 배경 위).

### Android 의 translucent
```tsx
<StatusBar translucent backgroundColor="transparent" />
```
→ 상태바 영역이 투명. 우리 view 가 그 아래까지 채움. SafeArea 와 결합 필수.

## 6. expo-status-bar — 더 간단

```bash
# Expo 또는 bare 둘 다 가능
yarn add expo-status-bar
```

```tsx
import { StatusBar } from 'expo-status-bar';
<StatusBar style="dark" />
```

→ iOS / Android 모두 통일된 API.

## 7. 가장 흔한 layout 패턴

### (A) 전체 페이지 SafeArea
```tsx
function MyPage() {
  return (
    <SafeAreaView style={{ flex: 1, backgroundColor: 'white' }}>
      <StatusBar barStyle="dark-content" backgroundColor="white" />
      <Header />
      <ScrollView>...</ScrollView>
    </SafeAreaView>
  );
}
```

### (B) 헤더 만 SafeArea (full bleed body)
```tsx
function MyPage() {
  return (
    <View style={{ flex: 1 }}>
      <SafeAreaView edges={['top']} style={{ backgroundColor: 'blue' }}>
        <Header />        {/* 헤더가 노치 + 상태바 영역까지 자연스럽게 */}
      </SafeAreaView>
      <ScrollView>...</ScrollView>
    </View>
  );
}
```

### (C) Background 가 다른 색 (full bleed)
```tsx
function MyPage() {
  const insets = useSafeAreaInsets();
  return (
    <View style={{ flex: 1, backgroundColor: 'blue' }}>
      <StatusBar barStyle="light-content" backgroundColor="blue" translucent />
      <View style={{ paddingTop: insets.top }}>
        <Header />
      </View>
      <View style={{ flex: 1, paddingBottom: insets.bottom }}>
        ...
      </View>
    </View>
  );
}
```

## 8. react-navigation 과의 결합

react-navigation v7 의 header 는 **자동으로 SafeArea 적용**. 따로 신경 X.

```tsx
// 헤더가 있는 화면
<Stack.Screen name="Home" component={Home} options={{ title: '홈' }} />

// 헤더 없는 화면 — SafeArea 직접
<Stack.Screen name="Custom" component={Custom} options={{ headerShown: false }} />

function Custom() {
  return (
    <SafeAreaView style={{ flex: 1 }}>
      ...
    </SafeAreaView>
  );
}
```

→ `headerShown: false` 일 때만 SafeAreaView.

## 9. KeyboardAvoidingView — 키보드 회피

```tsx
import { KeyboardAvoidingView, Platform } from 'react-native';

<KeyboardAvoidingView
  behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
  style={{ flex: 1 }}
>
  <ScrollView keyboardShouldPersistTaps="handled">
    <TextInput ... />
  </ScrollView>
</KeyboardAvoidingView>
```

### behavior
- iOS: `padding` (대부분).
- Android: `height` 또는 `undefined` (Android 가 보통 자동 처리).

### keyboardVerticalOffset
헤더 / 상단 요소만큼 보정:
```tsx
<KeyboardAvoidingView
  behavior="padding"
  keyboardVerticalOffset={headerHeight}
>
```

## 10. Splash Screen — 앱 시작 화면

`react-native-splash-screen`:
```bash
yarn add react-native-splash-screen
```

```tsx
import SplashScreen from 'react-native-splash-screen';

useEffect(() => {
  SplashScreen.hide();    // JS 준비되면 숨김
}, []);
```

→ native splash 이미지는 별도 설정 ([[../configuration|configuration]]).

## 11. job-answer-app-rn 의 SafeArea 패턴

```tsx
// App.tsx
<SafeAreaProvider>
  <NavigationContainer>...</NavigationContainer>
</SafeAreaProvider>

// 각 페이지 — 헤더 없는 경우
<SafeAreaView edges={['top']} style={{ flex: 1, backgroundColor: 'white' }}>
  ...
</SafeAreaView>

// 또는 페이지 내 ProgressHeader 가 자체 SafeArea 처리
```

## 12. 함정

1. **`react-native` 의 SafeAreaView 사용** — iOS 만 동작. `react-native-safe-area-context` 권장.
2. **SafeAreaProvider 누락** — `useSafeAreaInsets()` 가 0 return.
3. **StatusBar backgroundColor on iOS** — 무시됨. iOS 는 barStyle 만.
4. **edges 미지정 + 하단 탭** — 탭 위에 padding 추가 → 디자인 깨짐.
5. **Android translucent + 누락** — 상태바 색 안 보임 / 충돌.
6. **회전 / 화면 크기 변경** — Dimensions 재측정 필요. useWindowDimensions.
7. **iPad** — SafeArea 가 0 일 수 있음. 정상.

## 13. 다음 단계

- [[../navigation/navigation]] — 페이지 이동
- [[../performance/flatlist]] — 큰 list

## 14. 외부 자료

- [react-native-safe-area-context](https://github.com/th3rdwave/react-native-safe-area-context)
- [reactnative.dev — KeyboardAvoidingView](https://reactnative.dev/docs/keyboardavoidingview)
- [reactnative.dev — StatusBar](https://reactnative.dev/docs/statusbar)

## 15. 관련

- [[core-concepts]]
- [[platform-differences]]
- [[../navigation/navigation]]
