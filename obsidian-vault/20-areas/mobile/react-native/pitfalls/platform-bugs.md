---
title: "Platform Bugs — iOS vs Android 동작 차이"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, ios, android, platform-bugs]
---

# Platform Bugs

**[[pitfalls|↑ Pitfalls Hub]]**

> "iOS 만 작동 / Android 만 작동" 의 자주 만나는 케이스.

## 1. 키보드

### iOS 만 자동 처리
```tsx
<KeyboardAvoidingView behavior={Platform.OS === 'ios' ? 'padding' : 'height'}>
```

→ Android 는 보통 `windowSoftInputMode="adjustResize"` (AndroidManifest) 로 자동.

### Android 에서 input focus 시 화면 안 올라옴
`AndroidManifest.xml`:
```xml
<activity
  android:windowSoftInputMode="adjustResize">
```

## 2. shadow

### iOS — shadow*
```tsx
{ shadowColor: '#000', shadowOpacity: 0.1, shadowOffset: { width: 0, height: 2 }, shadowRadius: 4 }
```

### Android — elevation
```tsx
{ elevation: 4 }
```

→ 둘 다 필요. Platform.select 또는 두 prop 모두.

## 3. 배경 색

### iOS 의 `backgroundColor: 'transparent'`
- 동작.

### Android 의 transparent + elevation
- elevation 이 background 없는 element 에 안 보임. backgroundColor 명시 필요.

## 4. status bar

### iOS
- StatusBar 의 backgroundColor 무시.
- barStyle 만 적용.

### Android
- backgroundColor 적용.
- translucent prop 가능.

```tsx
{Platform.OS === 'android' && <StatusBar backgroundColor="white" />}
<StatusBar barStyle="dark-content" />
```

## 5. ripple effect

### Android — Pressable 의 `android_ripple`
```tsx
<Pressable android_ripple={{ color: 'rgba(0,0,0,0.1)' }} style={...}>
```

### iOS — opacity 변화 (`pressed`)
```tsx
<Pressable style={({ pressed }) => [styles.button, pressed && { opacity: 0.7 }]}>
```

→ Platform 별로 다르게 처리. 또는 `Touchable*` 류.

## 6. 폰트

### iOS 의 fontFamily
- PostScript name (`Pretendard-Bold` 또는 `Pretendard`).

### Android 의 fontFamily
- 파일 이름 (`Pretendard-Bold`).

→ 둘 다 동일하게 명명한 폰트 파일 권장.

### fontWeight 의 iOS / Android 차이
- iOS: `'600'`, `'700'`, `'900'` 등 number string + native 폰트 weight.
- Android: `'normal'` / `'bold'` 외에는 일부 무시. 정확한 weight 원하면 fontFamily 별로 분리 (`Pretendard-Bold`, `Pretendard-Medium`).

## 7. Modal

### iOS 의 transparent Modal
- 자동 동작.

### Android — `<Modal>` 의 statusBarTranslucent
```tsx
<Modal transparent statusBarTranslucent>
```

→ Android 의 statusbar 영역 침범 옵션.

## 8. SafeArea

### iPhone 의 notch / 홈바
- `SafeAreaView` (from `react-native-safe-area-context`) 자동.

### Android
- 노치 디바이스가 있지만 일반적으로 SafeArea = 0.
- statusbar 만 신경.

## 9. FlatList 의 onEndReached

### Android
- 처음 mount 시 onEndReached 가 잘못 firing 하는 경우.

해결:
```tsx
const onEndReached = useCallback(() => {
  if (hasNextPage && !isFetchingNextPage && data?.pages.length > 0) {
    fetchNextPage();
  }
}, [hasNextPage, isFetchingNextPage, data]);
```

→ 데이터 있을 때만 trigger.

## 10. Linking — 외부 앱

### iOS 의 `LSApplicationQueriesSchemes`
`Info.plist`:
```xml
<key>LSApplicationQueriesSchemes</key>
<array>
  <string>tel</string>
  <string>mailto</string>
  <string>kakaokompassauth</string>
</array>
```

→ Linking.canOpenURL 이 false return 시 누락 확인.

### Android — 자동.

## 11. Background — Long-running tasks

### iOS
- background mode 등록 (Background fetch / Audio / VoIP).
- 시간 제한 (보통 30초~).

### Android
- foreground service.
- 권한 (`FOREGROUND_SERVICE_*`).

→ RN 만으론 부족. native module 또는 expo-task-manager.

## 12. Push 알림

### iOS
- APNs.
- `provisional` 권한 (조용한 알림 — 사용자 동의 없이).
- 시뮬레이터 푸시 = iOS 16+ 의 `.apns` drop.

### Android
- FCM.
- POST_NOTIFICATIONS 권한 (13+).
- 채널 (8+).

자세히 [[../native-features/push-fcm]].

## 13. 시스템 알림 / dialog

### iOS — Alert
```ts
Alert.alert('제목', '메시지', [{ text: '확인' }]);
```

### Android — Alert + ToastAndroid
```ts
ToastAndroid.show('메시지', ToastAndroid.SHORT);   // Android-only
```

→ Cross-platform toast = `react-native-toast-message`.

## 14. Date / Time

### iOS DatePicker
- 모달 / 인라인 / wheels.

### Android DatePicker
- 다이얼로그.

→ `@react-native-community/datetimepicker` 의 UI 가 다름. 또는 자체 BottomSheet.

## 15. text wrap 차이

### iOS 의 `<Text>` 안 자동 wrap
- 기본 동작.

### Android 의 word-wrap
- 영문 / 한글 mix 시 줄바꿈 다름. line-height 조정.

## 16. 색 표현

iOS 의 hex 색 vs Android 의 8-digit hex:
```tsx
{ backgroundColor: '#0064FFAA' }    // 67% 투명도
```

→ 둘 다 지원. 일관성.

## 17. localhost (Metro)

- iOS sim: `localhost` OK.
- Android emulator: **`10.0.2.2`**.
- 실기기: PC 의 LAN IP.

```ts
const HOST = Platform.OS === 'android' ? '10.0.2.2' : 'localhost';
const api = axios.create({ baseURL: `http://${HOST}:8080` });
```

## 18. Bundle 크기

### iOS 의 App Thinning
- 자동. 각 디바이스에 맞는 asset 만.

### Android 의 App Bundle (.aab)
- Play Store 가 device 별 split.

→ 둘 다 활성화 시 size 절약.

## 19. 흔한 trick

```tsx
// Platform 별 inline 한 줄
const value = Platform.OS === 'ios' ? 16 : 24;

// 객체 + select
const styles = StyleSheet.create({
  container: {
    paddingTop: Platform.select({ ios: 0, android: 24 }) ?? 0,
  },
});

// 파일 별 분기 — Button.ios.tsx / Button.android.tsx
import { Button } from './Button';   // 자동 platform 선택
```

## 20. 함정

1. **iOS 만 테스트** — Android 에서 elevation / windowSoftInput 누락.
2. **Android 만 테스트** — iOS 의 노치 / shadow 누락.
3. **`Platform.Version`** — iOS = string, Android = number.
4. **시뮬레이터 ≠ 실기기** — push / camera / OAuth 차이.
5. **iPad / 가로 모드 / tablet** — 디자인 깨짐.
6. **다른 OS version 의 호환성** — iOS 13 / Android 8.

## 21. 외부 자료

- [reactnative.dev — Platform-Specific Code](https://reactnative.dev/docs/platform-specific-code)
- [Android KeyboardAvoiding](https://stackoverflow.com/q/29313156)

## 22. 관련

- [[pitfalls]]
- [[metro-cache]]
- [[../core-concepts/platform-differences]]
