---
title: "Platform Differences — Platform.OS / .ios.tsx / .android.tsx"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, platform, ios, android]
---

# Platform Differences

**[[core-concepts|↑ Core Concepts]]**

> RN 은 한 코드로 두 OS. **공통 99% + iOS / Android 의 1% 분기**.

## 1. Platform.OS — 런타임 분기

```tsx
import { Platform } from 'react-native';

if (Platform.OS === 'ios') {
  // iOS 만
} else if (Platform.OS === 'android') {
  // Android 만
}

console.log(Platform.OS);          // 'ios' | 'android'
console.log(Platform.Version);     // iOS: '17.0'  | Android: 34 (API)
```

### `Platform.select` — 객체 분기

```tsx
const styles = StyleSheet.create({
  container: {
    padding: 16,
    ...Platform.select({
      ios: { shadowColor: '#000', shadowOpacity: 0.1 },
      android: { elevation: 4 },
    }),
  },
});
```

## 2. 파일 분기 — `.ios.tsx` / `.android.tsx`

```
src/components/Button/
├── Button.ios.tsx
├── Button.android.tsx
└── Button.tsx               # 둘 다 안 맞으면 fallback
```

```tsx
// 사용
import { Button } from './Button';     // OS 별로 자동 선택
```

→ 둘의 native 컴포넌트가 너무 다를 때. (예: iOS 의 BlurView vs Android 의 다른 구현)

→ `metro.config.js` 의 `resolver.sourceExts` 에 `.ios`, `.android` 가 자동 포함.

## 3. iOS / Android 의 흔한 차이

### 키보드 회피 동작

```tsx
<KeyboardAvoidingView
  behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
>
  ...
</KeyboardAvoidingView>
```

→ iOS = padding, Android = height (또는 무 처리).

### 상태바

```tsx
import { StatusBar, Platform } from 'react-native';

<StatusBar
  barStyle={Platform.OS === 'ios' ? 'dark-content' : 'light-content'}
  backgroundColor="#0064FF"      // Android 만 적용
/>
```

→ Android 는 backgroundColor 지원, iOS 는 무시.

### 폰트

```tsx
fontFamily: Platform.select({
  ios: 'Helvetica',                       // PostScript name
  android: 'sans-serif',                  // 시스템 폰트 이름
}),
```

### 그림자

```tsx
...Platform.select({
  ios: {
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
  },
  android: {
    elevation: 4,                          // android only
  },
}),
```

### Ripple effect (Android 의 터치 ripple)

```tsx
import { Platform, Pressable } from 'react-native';

<Pressable
  android_ripple={{ color: 'rgba(0,0,0,0.1)', borderless: false }}
  style={({ pressed }) => Platform.OS === 'ios' && pressed && { opacity: 0.7 }}
>
  ...
</Pressable>
```

### Toast

```tsx
import { Platform, ToastAndroid, Alert } from 'react-native';

const showToast = (msg: string) => {
  if (Platform.OS === 'android') {
    ToastAndroid.show(msg, ToastAndroid.SHORT);
  } else {
    Alert.alert(msg);
    // 또는 react-native-toast-message 같은 lib 통일
  }
};
```

### 권한

```tsx
import { Platform, PermissionsAndroid } from 'react-native';
import { request, PERMISSIONS } from 'react-native-permissions';

if (Platform.OS === 'android') {
  await PermissionsAndroid.request(PermissionsAndroid.PERMISSIONS.CAMERA);
} else {
  await request(PERMISSIONS.IOS.CAMERA);
}
```

→ `react-native-permissions` 가 통일된 API 제공. 자세히 [[../native-features/permissions]].

### Linking — 외부 앱

```tsx
// 전화
Linking.openURL('tel:01012345678');

// 메일
Linking.openURL('mailto:hello@example.com');

// SMS
Linking.openURL(Platform.OS === 'ios' ? 'sms:01012345678' : 'sms:01012345678?body=hi');
```

### back button (Android)

```tsx
import { BackHandler } from 'react-native';

useEffect(() => {
  if (Platform.OS !== 'android') return;
  const sub = BackHandler.addEventListener('hardwareBackPress', () => {
    // true return = back 동작 가로채기
    return true;
  });
  return () => sub.remove();
}, []);
```

→ iOS 는 hardware back X (홈 인디케이터 또는 화면 내 버튼).

### Date / Time picker

→ native lib 마다 iOS vs Android UI 가 완전 다름. `@react-native-community/datetimepicker` 등.

## 4. 디자인 차이 — Material vs Human Interface

| | iOS (Apple HIG) | Android (Material) |
| --- | --- | --- |
| 헤더 | center title | left title |
| navigation | swipe back (왼쪽 끝) | hardware back |
| 폰트 | SF / Pretendard | Roboto / Pretendard |
| 그림자 | 약함 | 강함 (elevation) |
| 색 | 부드러움 | 진함 |
| 모달 | bottom sheet 흔함 | dialog 흔함 |
| 액션 | 오른쪽 (CTA) | 오른쪽 (FAB) |

→ 한국 앱은 iOS-style 통일이 흔함 (디자이너 기준). Material 적용은 Android-only 영역에만.

## 5. iOS / Android 별 디바이스 변수

```tsx
import { Platform, StatusBar, Dimensions } from 'react-native';

// iOS 의 노치 / Dynamic Island
const isNotch = Platform.OS === 'ios' && height >= 812;
const isDynamicIsland = Platform.OS === 'ios' && /* iPhone 14 Pro 이상 */;

// Android 상태바 높이
const statusBarHeight = Platform.OS === 'android' ? StatusBar.currentHeight ?? 0 : 0;
```

→ 더 정확한 방법: [[safe-area-and-status-bar]] 의 SafeAreaView.

## 6. 빌드 시 분기

```js
// metro.config.js — RN bundler 의 platform 인식
const config = {
  resolver: {
    sourceExts: [...defaultConfig.resolver.sourceExts, 'tsx', 'ts'],
  },
};
```

`.ios.tsx`, `.android.tsx` 우선순위:
- iOS 빌드 시: `Button.ios.tsx` > `Button.tsx`.
- Android 빌드 시: `Button.android.tsx` > `Button.tsx`.

## 7. native module — 한쪽만 지원

```tsx
// 예: iOS Apple SignIn 만 native 지원
import { Platform } from 'react-native';
import appleAuth from '@invertase/react-native-apple-authentication';

if (Platform.OS === 'ios') {
  await appleAuth.performRequest({ ... });
} else {
  // Web flow 또는 다른 방식
}
```

→ Apple Sign In 은 iOS 13+ 만 native. Android 는 web view fallback 또는 미지원.

## 8. job-answer-app-rn 의 분기 예 (추정)

```tsx
// SocialLoginSection
{Platform.OS === 'ios' && <AppleLoginButton />}
<KakaoLoginButton />
<GoogleLoginButton />
```

```tsx
// StatusBar
<StatusBar
  barStyle="dark-content"
  backgroundColor="white"      // Android 만
  translucent={Platform.OS === 'android'}
/>
```

```tsx
// Header padding
{
  paddingTop: Platform.OS === 'ios' ? safeAreaInsets.top : StatusBar.currentHeight,
}
```

## 9. testing — 두 플랫폼 모두 확인

- 시뮬레이터 / 에뮬레이터로 둘 다 정기 확인.
- 한 OS 만 테스트 시 다른 OS 에서 깨지는 케이스 빈번.
- CI 의 양쪽 빌드 가능 ([[../deployment/deployment]]).

## 10. 함정

1. **iOS 만 테스트** — Android 에서 elevation 누락 → 그림자 안 보임.
2. **Android 만 테스트** — iOS 의 노치 침범.
3. **`Platform.Version`** — iOS 는 string, Android 는 number. 비교 시 타입 변환.
4. **`.ios.tsx` 의존성 누락** — Button.ios.tsx 만 있고 Button.android.tsx 없으면 Android fallback 필요.
5. **권한 메시지 한쪽만** — iOS Info.plist, Android Manifest 둘 다.
6. **back button 미처리** — Android 사용자가 의도치 않게 앱 종료.
7. **DateTimePicker UI 충격** — 모달 vs 인라인 다른 디자인. lib 의 옵션 확인.

## 11. 외부 자료

- [reactnative.dev — Platform-Specific Code](https://reactnative.dev/docs/platform-specific-code)
- [Apple HIG](https://developer.apple.com/design/human-interface-guidelines/)
- [Material Design](https://m3.material.io/)

## 12. 다음 단계

- [[safe-area-and-status-bar]] — SafeAreaView / StatusBar

## 13. 관련

- [[core-concepts]]
- [[../native-features/permissions]]
- [[safe-area-and-status-bar]]
