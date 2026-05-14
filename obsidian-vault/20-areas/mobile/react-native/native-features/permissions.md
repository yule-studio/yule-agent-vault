---
title: "Permissions — react-native-permissions"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, permissions, ios, android]
---

# Permissions

**[[native-features|↑ Native Features Hub]]**

> iOS + Android 의 권한을 통일된 API 로. job-answer-app-rn 의 표준 라이브러리.

## 1. 설치

```bash
yarn add react-native-permissions
cd ios && pod install
```

## 2. iOS — `Info.plist` 의 메시지 (필수)

```xml
<!-- 카메라 -->
<key>NSCameraUsageDescription</key>
<string>프로필 사진 촬영을 위해 카메라가 필요합니다.</string>

<!-- 마이크 -->
<key>NSMicrophoneUsageDescription</key>
<string>음성 답변을 위해 마이크가 필요합니다.</string>

<!-- 갤러리 -->
<key>NSPhotoLibraryUsageDescription</key>
<string>사진 첨부를 위해 사진 접근이 필요합니다.</string>
<key>NSPhotoLibraryAddUsageDescription</key>
<string>저장하기 위해 사진 추가 권한이 필요합니다.</string>

<!-- 위치 -->
<key>NSLocationWhenInUseUsageDescription</key>
<string>주변 정보를 위해 위치가 필요합니다.</string>

<!-- 알림 — 자동 (require 시 시스템 dialog) -->
```

→ **메시지 누락 시 App Store 리젝**. 한국어 + 영어.

## 3. Android — `AndroidManifest.xml`

```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.RECORD_AUDIO" />
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"
                 android:maxSdkVersion="28" />     <!-- legacy -->
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />     <!-- Android 13+ -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
```

→ Manifest 의 permission 은 declare. 실제 요청은 코드.

## 4. Podfile — iOS 의 permission lib

`ios/Podfile`:
```ruby
permissions_path = '../node_modules/react-native-permissions/ios'

pod 'Permission-Camera', :path => "#{permissions_path}/Camera"
pod 'Permission-Microphone', :path => "#{permissions_path}/Microphone"
pod 'Permission-PhotoLibrary', :path => "#{permissions_path}/PhotoLibrary"
pod 'Permission-Notifications', :path => "#{permissions_path}/Notifications"
# 필요한 권한만
```

→ 사용 안 하는 권한 pod 은 안 받기 (bundle 절약).

또는 `package.json`:
```json
"reactNativePermissionsIOS": [
  "Camera",
  "Microphone",
  "PhotoLibrary",
  "Notifications"
]
```

→ `pod install` 시 자동 적용.

## 5. 사용 — check / request

```tsx
import { check, request, PERMISSIONS, RESULTS } from 'react-native-permissions';
import { Platform } from 'react-native';

const cameraPermission = Platform.select({
  ios: PERMISSIONS.IOS.CAMERA,
  android: PERMISSIONS.ANDROID.CAMERA,
})!;

async function checkAndRequest() {
  const status = await check(cameraPermission);
  // 'unavailable' | 'denied' | 'blocked' | 'granted' | 'limited'

  if (status === RESULTS.DENIED) {
    // 한 번도 안 물어봄 — request
    const newStatus = await request(cameraPermission);
    return newStatus === RESULTS.GRANTED;
  }
  if (status === RESULTS.GRANTED || status === RESULTS.LIMITED) {
    return true;
  }
  if (status === RESULTS.BLOCKED) {
    // 영구 거부 — Settings 로 안내
    Alert.alert('권한 필요', '설정에서 카메라 권한을 켜주세요.', [
      { text: '취소', style: 'cancel' },
      { text: '설정 열기', onPress: () => Linking.openSettings() },
    ]);
    return false;
  }
  return false;
}
```

### 결과
- `unavailable` — 디바이스에 기능 없음 (예: 시뮬레이터의 카메라).
- `denied` — 아직 안 물어봤거나 한 번 거부 (iOS 만 재요청 가능, Android 1 회만).
- `blocked` — 영구 거부 (iOS: deny + don't ask again).
- `granted` — 허용.
- `limited` — 부분 허용 (iOS 의 사진 선택만).

## 6. multiple — 여러 권한 한 번에

```tsx
import { requestMultiple, PERMISSIONS, RESULTS } from 'react-native-permissions';

const statuses = await requestMultiple([
  PERMISSIONS.IOS.CAMERA,
  PERMISSIONS.IOS.MICROPHONE,
]);

if (
  statuses[PERMISSIONS.IOS.CAMERA] === RESULTS.GRANTED &&
  statuses[PERMISSIONS.IOS.MICROPHONE] === RESULTS.GRANTED
) {
  // 둘 다 OK
}
```

## 7. POST_NOTIFICATIONS — Android 13+

```ts
if (Platform.OS === 'android' && Platform.Version >= 33) {
  await request(PERMISSIONS.ANDROID.POST_NOTIFICATIONS);
}
```

→ Android 12 이하는 자동 허용. 13+ 만 명시 필요.

## 8. 사용 시점 — UX

```
권한 요청 = 사용자가 그 기능을 쓰려고 누른 직후
```

- 앱 시작 시 모든 권한 요청 ❌ — 거부율 높음.
- "사진 첨부" 버튼 누름 → 그 직후 photo library 요청 ✅.

### preflight UI 패턴
```tsx
// 1. 앱이 먼저 "왜" 설명 UI 표시
<Modal>
  <Text>사진 첨부를 위해 갤러리 권한이 필요합니다.</Text>
  <Button title="허용" onPress={async () => {
    setModalOff();
    await request(PERMISSIONS.IOS.PHOTO_LIBRARY);
  }} />
</Modal>
```

→ 사용자가 시스템 dialog 마주치기 전 우리 설명을 본다 → 허용율 향상.

## 9. 권한 별 코드 매핑

```ts
const map = {
  camera: Platform.select({
    ios: PERMISSIONS.IOS.CAMERA,
    android: PERMISSIONS.ANDROID.CAMERA,
  })!,
  microphone: Platform.select({
    ios: PERMISSIONS.IOS.MICROPHONE,
    android: PERMISSIONS.ANDROID.RECORD_AUDIO,
  })!,
  photo: Platform.select({
    ios: PERMISSIONS.IOS.PHOTO_LIBRARY,
    android: PERMISSIONS.ANDROID.READ_MEDIA_IMAGES,
  })!,
  location: Platform.select({
    ios: PERMISSIONS.IOS.LOCATION_WHEN_IN_USE,
    android: PERMISSIONS.ANDROID.ACCESS_FINE_LOCATION,
  })!,
};
```

## 10. job-answer-app-rn 패턴

```tsx
// hooks/usePermission.ts
export function usePermission(type: 'camera' | 'microphone' | 'photo') {
  const permission = useMemo(() => map[type], [type]);

  const ensure = async () => {
    const status = await check(permission);
    if (status === RESULTS.GRANTED || status === RESULTS.LIMITED) return true;
    if (status === RESULTS.DENIED) {
      const next = await request(permission);
      return next === RESULTS.GRANTED;
    }
    if (status === RESULTS.BLOCKED) {
      Alert.alert('권한 필요', '설정에서 권한을 켜주세요.', [
        { text: '취소', style: 'cancel' },
        { text: '설정', onPress: () => Linking.openSettings() },
      ]);
      return false;
    }
    return false;
  };

  return { ensure };
}

// 사용
const { ensure } = usePermission('microphone');
const onRecord = async () => {
  if (!(await ensure())) return;
  // 녹음 시작
};
```

## 11. 함정

1. **Info.plist 의 메시지 누락** — iOS 첫 요청 시 강제 종료.
2. **Manifest 의 권한 누락** — Android 가 silent 거부.
3. **Android 의 hard limit** — 한 번 거부 후 자동 BLOCKED. 시스템 설정 안내.
4. **Podfile 의 permission pod 누락** — `Permission-X is undefined`.
5. **앱 시작 시 일괄 요청** — UX 안 좋음 + 거부율 높음.
6. **권한 BLOCKED 상태에서 다시 request** — denied 그대로 return. Settings 안내가 정답.
7. **iOS 14+ 의 사진 limited** — 사용자가 일부 사진만 허용. limited 도 사용 가능.

## 12. 외부 자료

- [react-native-permissions](https://github.com/zoontek/react-native-permissions)
- [iOS Permission Strings](https://developer.apple.com/library/archive/documentation/General/Reference/InfoPlistKeyReference/Articles/CocoaKeys.html)

## 13. 관련

- [[native-features]]
- [[image-picker]]
- [[voice-stt]]
- [[push-fcm]]
