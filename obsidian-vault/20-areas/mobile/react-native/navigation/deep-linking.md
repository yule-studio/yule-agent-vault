---
title: "Deep Linking — URL 스킴 + Universal Link / App Link"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, deep-linking, universal-link, app-link]
---

# Deep Linking

**[[navigation|↑ Navigation Hub]]**

> 외부 URL 로 앱의 특정 화면을 열기. 카카오 로그인 콜백, 이메일 링크 등 필수.

## 1. 종류

| | URL 형태 | 비고 |
| --- | --- | --- |
| **URL 스킴** | `myapp://path` | 가장 단순. 사용자 앱 설치 없으면 동작 X. |
| **iOS Universal Link** | `https://myapp.com/path` | https URL. 앱 없으면 사파리 fallback. |
| **Android App Link** | `https://myapp.com/path` | 위와 동일. autoVerify 권장. |

→ 신규 = Universal Link + App Link 권장. 단 설정 복잡.

## 2. URL 스킴 — 가장 단순

### iOS — Info.plist
```xml
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleURLName</key>
    <string>com.example.myapp</string>
    <key>CFBundleURLSchemes</key>
    <array>
      <string>myapp</string>
    </array>
  </dict>
</array>
```

### Android — AndroidManifest.xml
```xml
<activity
  android:name=".MainActivity"
  android:launchMode="singleTask"
  android:exported="true">
  <intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="myapp" />
  </intent-filter>
</activity>
```

### 테스트
```bash
# iOS sim
xcrun simctl openurl booted "myapp://detail/5"

# Android emulator
adb shell am start -W -a android.intent.action.VIEW -d "myapp://detail/5"
```

## 3. react-navigation 의 linking 설정

```tsx
// App.tsx
const linking = {
  prefixes: ['myapp://', 'https://myapp.com'],
  config: {
    screens: {
      Home: '',                                    // myapp://
      Detail: 'detail/:id',                        // myapp://detail/5
      Settings: {
        screens: {
          Account: 'settings/account',
        },
      },
      Auth: {
        screens: {
          KakaoCallback: 'auth/kakao',             // OAuth 콜백
        },
      },
    },
  },
};

<NavigationContainer linking={linking}>
  ...
</NavigationContainer>
```

→ 외부 URL → 자동 navigate 매핑.

## 4. 코드에서 URL 처리

```tsx
import { Linking } from 'react-native';

// 외부 URL 열기
Linking.openURL('https://google.com');
Linking.openURL('myapp://detail/5');

// 우리 앱이 받은 URL 처리
useEffect(() => {
  // 1. 앱이 background 상태로 켜져 있을 때
  const sub = Linking.addEventListener('url', ({ url }) => {
    handleDeepLink(url);
  });

  // 2. 앱이 꺼져 있다가 link 로 시작
  Linking.getInitialURL().then((url) => {
    if (url) handleDeepLink(url);
  });

  return () => sub.remove();
}, []);

function handleDeepLink(url: string) {
  console.log(url);  // 'myapp://detail/5'
  // react-navigation 의 linking 이 자동 처리. 직접 처리도 가능.
}
```

## 5. Universal Link (iOS)

장점: https URL → 앱 설치 시 앱, 없으면 사파리.
설정 (복잡):

1. Apple Developer — App ID 의 Associated Domains 활성화.
2. Xcode — Signing & Capabilities → Associated Domains 추가:
   ```
   applinks:myapp.com
   ```
3. 서버 — `https://myapp.com/.well-known/apple-app-site-association` 파일:
   ```json
   {
     "applinks": {
       "details": [{
         "appIDs": ["TEAMID.com.example.myapp"],
         "components": [
           { "/": "/detail/*" },
           { "/": "/auth/*" }
         ]
       }]
     }
   }
   ```
4. 빌드 + 테스트.

## 6. App Link (Android)

1. AndroidManifest:
```xml
<intent-filter android:autoVerify="true">
  <action android:name="android.intent.action.VIEW" />
  <category android:name="android.intent.category.DEFAULT" />
  <category android:name="android.intent.category.BROWSABLE" />
  <data
    android:scheme="https"
    android:host="myapp.com"
    android:pathPrefix="/detail" />
</intent-filter>
```

2. 서버 — `https://myapp.com/.well-known/assetlinks.json`:
```json
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.example.myapp",
    "sha256_cert_fingerprints": ["..."]
  }
}]
```

## 7. OAuth 콜백 패턴

```tsx
// 카카오 로그인의 redirect_uri 가 myapp://auth/kakao 라면
const linking = {
  prefixes: ['myapp://'],
  config: {
    screens: {
      KakaoCallback: 'auth/kakao',
    },
  },
};

function KakaoCallback({ route }) {
  const { code } = route.params;
  useEffect(() => {
    // BE 에 code 전달 + token 받기
    api.post('/auth/kakao', { code }).then(...);
  }, []);
  return <Spinner />;
}
```

→ 자세히 [[../auth/kakao-login]].

### native lib 사용 시
`@react-native-seoul/kakao-login` 같은 native lib 는 deep link 우회 (native API 직접 호출). RN deep link 설정 불필요.

→ 일반 OAuth (Google web flow 등) = deep link 필요. native lib = 불필요.

## 8. 알림에서 진입 — push notification

```tsx
import messaging from '@react-native-firebase/messaging';

// 알림 클릭 → 앱 진입
useEffect(() => {
  // background 알림 클릭
  messaging().onNotificationOpenedApp((message) => {
    const screen = message.data?.screen;
    if (screen === 'detail') {
      navigation.navigate('Detail', { id: message.data.id });
    }
  });

  // 앱 꺼져 있던 상태에서 알림으로 시작
  messaging().getInitialNotification().then((message) => {
    if (message) handleMessage(message);
  });
}, []);
```

→ 자세히 [[../native-features/push-fcm]].

## 9. 디버깅

```bash
# iOS — URL 시뮬레이션
xcrun simctl openurl booted "myapp://detail/5"

# Android
adb shell am start -W -a android.intent.action.VIEW -d "myapp://detail/5" com.example.myapp

# react-navigation 의 linking config 디버그
import { Linking } from 'react-native';
Linking.getInitialURL().then(console.log);

# react-navigation 의 매핑 결과 확인 — devtools 의 navigator state
```

## 10. 다른 앱 열기

```ts
// 전화
Linking.openURL('tel:01012345678');

// 메일
Linking.openURL('mailto:hello@example.com?subject=Hi&body=...');

// SMS
Linking.openURL('sms:01012345678');

// 카카오톡 (앱별 URL 스킴)
Linking.openURL('kakaolink://...');

// 외부 브라우저 (시스템)
Linking.openURL('https://google.com');
// → 시스템 브라우저로

// 인앱 브라우저 (앱 안 열기)
import InAppBrowser from 'react-native-inappbrowser-reborn';
await InAppBrowser.open('https://google.com');
```

→ 자세히 [[../native-features/inappbrowser]].

## 11. 권한 / 보안

- URL 의 params 는 신뢰할 수 없음 — XSS / injection 방어.
- 인증 token 을 URL 에 담지 말 것 — log 에 남음.
- Universal Link 검증 — autoVerify + signing.

## 12. 함정

1. **URL 스킴 등록 후 빌드 안 함** — Info.plist / Manifest 변경은 native 빌드 필요.
2. **`launchMode="singleTask"`** 누락 — Android 가 새 instance 만들어 state 잃음.
3. **`Linking.getInitialURL` 누락** — 앱 꺼져 있을 때 들어온 URL 못 잡음.
4. **Universal Link 의 server side AASA** — wellknown 파일이 https + valid + uncached.
5. **kakao native lib + deep link 중복 등록** — 충돌.
6. **dev mode 의 reload** — deep link 가 metro reload 와 충돌하는 케이스. 실기기 테스트.

## 13. 다음 단계

- [[../auth/auth]] — OAuth 콜백
- [[../native-features/push-fcm]] — 알림 → 진입

## 14. 외부 자료

- [react-navigation — Deep Linking](https://reactnavigation.org/docs/deep-linking)
- [iOS Universal Links](https://developer.apple.com/documentation/xcode/supporting-universal-links-in-your-app)
- [Android App Links](https://developer.android.com/training/app-links)

## 15. 관련

- [[navigation]]
- [[../auth/kakao-login]]
- [[../native-features/inappbrowser]]
- [[../native-features/push-fcm]]
