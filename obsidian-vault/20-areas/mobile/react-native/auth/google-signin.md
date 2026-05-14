---
title: "Google Sign In — @react-native-google-signin/google-signin"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, auth, google, sign-in]
---

# Google Sign In

**[[auth|↑ Auth Hub]]**

> v16+ native SDK. iOS + Android 양쪽 지원.

## 1. 설치

```bash
yarn add @react-native-google-signin/google-signin
cd ios && pod install
```

## 2. Google Cloud Console 설정

1. **Google Cloud Console** (console.cloud.google.com) → 프로젝트 생성.
2. **APIs & Services → Credentials** → OAuth client ID 만들기.
3. iOS:
   - Application type: iOS.
   - Bundle ID 입력 → `iOSClientId` 생성됨.
   - `GoogleService-Info.plist` 또는 reversed client ID 메모.
4. Android:
   - Application type: Android.
   - Package name + **SHA-1 fingerprint** (debug + release).
5. Web (BE 검증용 / serverClientId):
   - Application type: Web application.
   - 이 client ID 가 `webClientId` (=`serverClientId`).

→ 보통 BE 가 검증 시 web client ID 의 `aud` 사용.

## 3. SHA-1 fingerprint 얻기

```bash
# debug
cd android
./gradlew signingReport

# 결과의 SHA-1: 줄을 Google Cloud Console 의 Android client 에 등록
```

→ release 키는 별도 (production keystore 의 SHA-1).

## 4. 초기화

```ts
import { GoogleSignin } from '@react-native-google-signin/google-signin';

GoogleSignin.configure({
  webClientId: 'YOUR_WEB_CLIENT_ID.apps.googleusercontent.com',  // BE 가 검증할 client_id
  iosClientId: 'YOUR_IOS_CLIENT_ID.apps.googleusercontent.com',
  offlineAccess: true,             // refresh token 받기
  scopes: ['email', 'profile'],
  forceCodeForRefreshToken: true,
});
```

→ App 시작 시 1 회 (App.tsx 의 useEffect).

## 5. 로그인

```ts
import { GoogleSignin, statusCodes } from '@react-native-google-signin/google-signin';

async function signInWithGoogle() {
  try {
    await GoogleSignin.hasPlayServices({ showPlayServicesUpdateDialog: true });

    const userInfo = await GoogleSignin.signIn();
    // userInfo.idToken — BE 에 전달
    // userInfo.user — { id, email, name, photo, ... }

    const { data } = await api.post('/auth/google', {
      idToken: userInfo.idToken,
      serverAuthCode: userInfo.serverAuthCode,
    });

    await tokenStorage.setAccess(data.accessToken);
    await tokenStorage.setRefresh(data.refreshToken);
    navigation.reset({ index: 0, routes: [{ name: 'MainTab' }] });
  } catch (e) {
    if (e.code === statusCodes.SIGN_IN_CANCELLED) {
      // 사용자 취소
    } else if (e.code === statusCodes.IN_PROGRESS) {
      // 이미 진행 중
    } else if (e.code === statusCodes.PLAY_SERVICES_NOT_AVAILABLE) {
      // Android: Google Play Service 없음
    } else {
      console.error(e);
    }
  }
}
```

## 6. v16 의 API 변화

```ts
// 옛 (v9-)
const { idToken } = await GoogleSignin.signIn();

// v16+
const response = await GoogleSignin.signIn();
if (response.type === 'success') {
  const { idToken, user } = response.data;
}
```

→ response 의 type 으로 분기. v 마다 확인.

## 7. 현재 사용자 / silent sign-in

```ts
// 앱 시작 시 — 이미 로그인 상태 확인
const isSignedIn = GoogleSignin.hasPreviousSignIn();

if (isSignedIn) {
  // 자동 로그인 — UI 없이
  const userInfo = await GoogleSignin.signInSilently();
  // 또는 GoogleSignin.getCurrentUser()
}
```

## 8. 로그아웃 / 권한 해제

```ts
// 로그아웃 — token revoke X
await GoogleSignin.signOut();

// 권한 완전 해제 (회원 탈퇴 시)
await GoogleSignin.revokeAccess();
```

## 9. UI — 공식 버튼

```tsx
import { GoogleSigninButton } from '@react-native-google-signin/google-signin';

<GoogleSigninButton
  size={GoogleSigninButton.Size.Wide}
  color={GoogleSigninButton.Color.Dark}
  onPress={signInWithGoogle}
/>
```

→ Google brand guideline 자동 준수. 직접 디자인은 가이드 확인.

## 10. iOS 추가 설정 — `Info.plist`

```xml
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleURLSchemes</key>
    <array>
      <!-- iOSClientId 의 reversed client ID -->
      <string>com.googleusercontent.apps.123456789-xxxxxxxxxxxxxxxx</string>
    </array>
  </dict>
</array>
```

→ 없으면 redirect 실패.

## 11. Android 추가 설정

- `android/build.gradle` 의 minSdk 21+.
- `google-services.json` 배치 (`@react-native-firebase` 와 충돌 X).

## 12. job-answer-app-rn 패턴

```tsx
// components/SocialButton/Google.tsx
import { GoogleSignin } from '@react-native-google-signin/google-signin';

GoogleSignin.configure({
  webClientId: WEB_CLIENT_ID,
  iosClientId: IOS_CLIENT_ID,
});

export const GoogleLoginButton = () => {
  const handle = async () => {
    try {
      await GoogleSignin.hasPlayServices();
      const userInfo = await GoogleSignin.signIn();
      const { accessToken, refreshToken } = await authApi.googleLogin({
        idToken: userInfo.idToken!,
      });
      await tokenStorage.setAccess(accessToken);
      navigation.reset({ index: 0, routes: [{ name: 'MainTab' }] });
    } catch (e) {
      if (e.code !== statusCodes.SIGN_IN_CANCELLED) Alert.alert('Google 로그인 실패');
    }
  };

  return <SocialButton provider="google" onPress={handle} />;
};
```

## 13. 흔한 에러

### `DEVELOPER_ERROR` (Android)
- SHA-1 fingerprint 가 Google Cloud Console 에 등록 안 됨.
- package name 불일치.
- `webClientId` 가 잘못됨.

해결:
1. `./gradlew signingReport` 로 SHA-1 확인.
2. Google Cloud Console 의 Android OAuth client 에 등록.
3. 앱 재빌드.

### `PLAY_SERVICES_NOT_AVAILABLE`
- 에뮬레이터에 Play Service 없음.
- Genymotion / 비공식 에뮬레이터에서 흔함.

해결: Google Play Service 포함된 에뮬레이터 (Pixel + Google API) 사용.

### iOS 에서 redirect 안 됨
- `Info.plist` 의 reversed client ID 누락.

## 14. 함정

1. **SHA-1 mismatch** — Android 의 가장 흔한 에러.
2. **`webClientId` vs `iosClientId`** — 다른 ID. 헷갈리지 말기.
3. **release 빌드의 SHA-1** — debug 만 등록하고 release 안 함 → release 빌드에서 fail.
4. **idToken null** — `webClientId` 누락 또는 잘못. `webClientId` 필수.
5. **iOS Info.plist** 의 reversed client ID 누락.
6. **`hasPlayServices` 누락** — Android 에서 unhandled error.
7. **scope 의 과다 요청** — 동의율 감소.

## 15. 외부 자료

- [@react-native-google-signin/google-signin](https://github.com/react-native-google-signin/google-signin)
- [Google Identity — iOS](https://developers.google.com/identity/sign-in/ios/start)
- [Google Identity — Android](https://developers.google.com/identity/sign-in/android/start)

## 16. 관련

- [[auth]]
- [[apple-signin]]
- [[kakao-login]]
