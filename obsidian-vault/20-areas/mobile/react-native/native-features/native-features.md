---
title: "Native Features — 권한 / 푸시 / 카메라 / 음성 / 브라우저 / i18n"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, native-features, hub]
---

# Native Features

**[[../react-native|↑ RN Hub]]**

> RN 의 진가 = native 기능. 권한 / 푸시 / 미디어 / 음성 / 브라우저 / 다국어.

## 1. 카테고리

| 영역 | 노트 |
| --- | --- |
| 권한 (카메라/마이크/위치/사진) | [[permissions]] |
| 푸시 알림 (FCM) | [[push-fcm]] |
| 이미지 / 카메라 | [[image-picker]] |
| 음성 인식 (STT) | [[voice-stt]] |
| 인앱 브라우저 | [[inappbrowser]] |
| 다국어 / 로캘 | [[localize-i18n]] |

## 2. job-answer-app-rn 의 native lib

```json
"@react-native-firebase/app": "^23.5.0",
"@react-native-firebase/messaging": "^23.5.0",
"@notifee/react-native": "^9.1.8",
"react-native-permissions": "^5.4.2",
"react-native-image-picker": "^8.2.1",
"@react-native-voice/voice": "^3.2.4",
"react-native-nitro-sound": "^0.2.10",
"react-native-inappbrowser-reborn": "^3.7.0",
"react-native-device-info": "^14.0.4",
"react-native-localize": "^3.5.2",
"react-native-clipboard/clipboard": "^1.16.3",
"react-native-channel-plugin": "^0.12.1",
"i18n-js": "^4.5.1"
```

→ 거의 모든 모바일 앱의 표준.

## 3. native lib 설치 공통 패턴

```bash
yarn add <native-lib>
cd ios && pod install        # iOS native dep
# Android — autolink (RN 0.69+ 자동)
```

→ 항상 pod install. autolink 가 대부분 처리.

### 추가 native 설정
- **Info.plist** — 권한 메시지.
- **AndroidManifest.xml** — 권한 + intent-filter.
- **AppDelegate** — 일부 lib (kakao 등) 의 callback 등록.
- **MainApplication.kt/.java** — package 등록 (드물게).

## 4. 권한 요청 — 기본 패턴

```tsx
import { request, PERMISSIONS, RESULTS } from 'react-native-permissions';
import { Platform } from 'react-native';

async function requestCameraPermission() {
  const permission = Platform.select({
    ios: PERMISSIONS.IOS.CAMERA,
    android: PERMISSIONS.ANDROID.CAMERA,
  })!;

  const status = await request(permission);

  if (status === RESULTS.GRANTED) {
    // 사용
  } else if (status === RESULTS.BLOCKED) {
    // 사용자가 영구 거부 — Settings 로 이동 권유
  }
}
```

자세히 [[permissions]].

## 5. 학습 우선순위

1. **[[permissions]]** — 모든 native 기능의 전제.
2. **[[push-fcm]]** — Firebase + native 설정 가장 복잡.
3. **[[image-picker]]** — 카메라 / 갤러리.
4. **[[localize-i18n]]** — 다국어 (한 + 영 + 일).
5. **[[inappbrowser]]** — 외부 link / OAuth web.
6. **[[voice-stt]]** — job-answer-app-rn 특화 (음성 답변).

## 6. native module 의 디버깅

- iOS 빌드 에러 → Xcode 열어 native log 확인.
- Android 빌드 에러 → `cd android && ./gradlew app:assembleDebug --info`.
- `pod install` 후 dependency 충돌 → Podfile.lock 검토.

## 7. 함정

1. **pod install 누락** — iOS 빌드 실패 또는 lib 가 undefined.
2. **권한 메시지 누락** — iOS 의 흔한 reject 이유.
3. **autolink 와 manual link 혼용** — duplicate symbol.
4. **lib version 충돌** — peer dep 확인.
5. **dev 환경 의 sandbox** — 푸시는 sandbox APNs vs production APNs.

## 8. 다음 단계

영역별 노트 참조 ↓

## 9. 관련

- [[permissions]]
- [[push-fcm]]
- [[image-picker]]
- [[voice-stt]]
- [[inappbrowser]]
- [[localize-i18n]]
