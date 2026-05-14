---
title: "Kakao Login — @react-native-seoul/kakao-login"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, auth, kakao, sign-in, korean]
---

# Kakao Login

**[[auth|↑ Auth Hub]]**

> 한국 시장 필수. job-answer-app-rn 의 핵심 lib (`@react-native-seoul/kakao-login` v5).

## 1. 설치

```bash
yarn add @react-native-seoul/kakao-login
cd ios && pod install
```

## 2. Kakao Developers 설정

1. **developers.kakao.com** → 내 애플리케이션 → 추가.
2. **앱 키** — Native App Key 메모.
3. **플랫폼** 등록:
   - iOS: Bundle ID (`com.example.myapp`).
   - Android: package name + 키 해시.
4. **카카오 로그인** 활성화 → Redirect URI 추가 (앱이므로 `kakao{appkey}://oauth`).
5. **동의항목** — 닉네임 / 이메일 / 프로필 등 받을 정보 설정.

## 3. Android 키 해시 등록

```bash
# debug
keytool -exportcert -alias androiddebugkey \
  -keystore ~/.android/debug.keystore | openssl sha1 -binary | openssl base64

# 결과 (예: `xxx=`) 를 Kakao Developers 의 Android 키해시 등록.
# release 키도 같은 방식.
```

## 4. iOS — `Info.plist`

```xml
<!-- URL scheme — 카카오 앱 callback -->
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleURLSchemes</key>
    <array>
      <string>kakao{NATIVE_APP_KEY}</string>     <!-- 예: kakao1a2b3c4d -->
    </array>
  </dict>
</array>

<!-- 카톡 앱 호출 허용 -->
<key>LSApplicationQueriesSchemes</key>
<array>
  <string>kakaokompassauth</string>
  <string>kakaolink</string>
</array>

<!-- 카카오 native app key -->
<key>KAKAO_APP_KEY</key>
<string>{NATIVE_APP_KEY}</string>
```

## 5. Android — `AndroidManifest.xml` + `build.gradle`

`AndroidManifest.xml`:
```xml
<activity
  android:name="com.kakao.sdk.auth.AuthCodeHandlerActivity"
  android:exported="true">
  <intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:host="oauth" android:scheme="kakao{NATIVE_APP_KEY}" />
  </intent-filter>
</activity>
```

`android/app/build.gradle`:
```gradle
android {
  defaultConfig {
    manifestPlaceholders = [
      KAKAO_APP_KEY: "{NATIVE_APP_KEY}"
    ]
  }
}
```

## 6. iOS — `AppDelegate.swift` (또는 `.mm`)

```swift
import RNKakaoLogins

// MARK: openURL
override func application(_ app: UIApplication, open url: URL, options: [UIApplication.OpenURLOptionsKey : Any] = [:]) -> Bool {
  if RNKakaoLogins.isKakaoTalkLoginUrl(url) {
    return RNKakaoLogins.handleOpenUrl(url)
  }
  return false
}
```

→ `.mm` (Objective-C++) 의 경우 lib README 참고.

## 7. 초기화

```ts
// App.tsx 또는 lib/auth/kakao.ts
import { initializeKakaoSDK } from '@react-native-seoul/kakao-login';
import { KAKAO_NATIVE_KEY } from '@env';

initializeKakaoSDK(KAKAO_NATIVE_KEY);
```

→ v5 부터 필요. 옛 버전은 자동.

## 8. 로그인

```ts
import { login, KakaoOAuthToken } from '@react-native-seoul/kakao-login';

async function signInWithKakao() {
  try {
    const token: KakaoOAuthToken = await login();
    // token.accessToken, refreshToken, idToken, expiresIn, ...

    const { data } = await api.post('/auth/kakao', {
      accessToken: token.accessToken,
      idToken: token.idToken,     // OIDC 활성화 시
    });

    await tokenStorage.setAccess(data.accessToken);
    await tokenStorage.setRefresh(data.refreshToken);
    navigation.reset({ index: 0, routes: [{ name: 'MainTab' }] });
  } catch (e) {
    Alert.alert('카카오 로그인 실패', e.message);
  }
}
```

→ 카카오톡 앱 설치돼 있으면 자동 카톡 앱으로 인증. 없으면 카카오 계정 web view.

## 9. 사용자 정보 가져오기

```ts
import { getProfile, KakaoProfile } from '@react-native-seoul/kakao-login';

const profile: KakaoProfile = await getProfile();
// profile.id, nickname, email, profileImageUrl, ageRange, gender, ...
```

→ BE 에서도 가능 (kakao accessToken 으로 카카오 API 호출). 보통 BE 가 처리.

## 10. 로그아웃 + 연결 해제

```ts
import { logout, unlink } from '@react-native-seoul/kakao-login';

// 로그아웃 — token 만 무효, 우리 앱과 연결은 유지
await logout();

// 연결 해제 — 회원 탈퇴. 우리 앱이 받은 권한 완전 revoke.
await unlink();
```

## 11. 자동 로그인 — 이미 로그인 상태 확인

```ts
import { getAccessToken } from '@react-native-seoul/kakao-login';

try {
  const tokenInfo = await getAccessToken();
  if (tokenInfo.accessToken) {
    // 자동 로그인 — 우리 token 갱신 시도
  }
} catch {
  // 로그인 안 됨
}
```

## 12. UI — 카카오 공식 버튼

카카오 가이드: 노란 배경 + 갈색 텍스트 (`#FEE500` / `#3C1E1E`).

```tsx
<Pressable
  onPress={signInWithKakao}
  style={{
    backgroundColor: '#FEE500',
    paddingVertical: 14,
    borderRadius: 6,
    flexDirection: 'row',
    justifyContent: 'center',
    alignItems: 'center',
  }}
>
  <KakaoIcon />
  <Text style={{ color: '#3C1E1E', fontWeight: '600', marginLeft: 8 }}>카카오 로그인</Text>
</Pressable>
```

→ 가이드: https://developers.kakao.com/docs/latest/ko/kakaologin/design-guide

## 13. BE 의 검증

BE 가 받은 `accessToken` / `idToken` 으로:
1. 카카오 API 호출: `GET https://kapi.kakao.com/v2/user/me` (`Authorization: Bearer ${accessToken}`).
2. 응답의 user id 로 우리 DB 의 user 찾기 / 생성.
3. 우리 JWT 발급.

또는 idToken (JWT) 의 signature 검증 (카카오의 JWKS endpoint).

## 14. job-answer-app-rn 패턴

```tsx
// components/SocialButton/Kakao.tsx
import { login, KakaoOAuthToken } from '@react-native-seoul/kakao-login';

export const KakaoLoginButton = () => {
  const handle = async () => {
    try {
      const t: KakaoOAuthToken = await login();
      const { accessToken, refreshToken } = await authApi.kakaoLogin({
        kakaoAccessToken: t.accessToken,
        idToken: t.idToken,
      });
      await tokenStorage.setAccess(accessToken);
      navigation.reset({ index: 0, routes: [{ name: 'MainTab' }] });
    } catch (e) {
      Alert.alert('카카오 로그인 실패');
    }
  };

  return <SocialButton provider="kakao" onPress={handle} />;
};
```

## 15. 함정

1. **앱 키 / 키 해시 불일치** — Android 의 흔한 에러.
2. **release vs debug 키 해시 별도** — production 빌드만 실패.
3. **Info.plist 의 LSApplicationQueriesSchemes** 누락 — 카톡 앱 호출 X.
4. **AppDelegate openURL 누락** — iOS 가 callback 받지 못함.
5. **카카오 앱 미설치 시** — 자동 web view fallback. UX 약간 다름.
6. **scope 추가 동의** — 처음 등록한 동의항목만. 추가 정보는 별도 동의 요청.
7. **dev 환경의 sandbox 사용자** — kakao 가 직접 제공 X. 진짜 카카오 계정 사용.
8. **email scope 의 동의 거부** — 사용자가 이메일 제공 거부 가능. null 처리.

## 16. 외부 자료

- [@react-native-seoul/kakao-login](https://github.com/crossplatformkorea/react-native-kakao-login)
- [Kakao Developers — REST API](https://developers.kakao.com/docs/latest/ko/kakaologin/rest-api)
- [Kakao Developers — RN SDK](https://developers.kakao.com/docs/latest/ko/kakaologin/react-native)

## 17. 관련

- [[auth]]
- [[apple-signin]]
- [[google-signin]]
