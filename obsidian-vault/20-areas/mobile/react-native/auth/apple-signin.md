---
title: "Apple Sign In — @invertase/react-native-apple-authentication"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, auth, apple, sign-in-with-apple, ios]
---

# Apple Sign In

**[[auth|↑ Auth Hub]]**

> iOS 의 native Sign in with Apple. **iOS-only** (Android 는 web fallback 또는 미지원).

## 1. 라이브러리

```bash
yarn add @invertase/react-native-apple-authentication
cd ios && pod install
```

## 2. Apple Developer 설정

1. **Apple Developer Console** → App ID 의 Capabilities → **Sign in with Apple** 활성화.
2. **Xcode** → Signing & Capabilities → **+ Capability** → Sign in with Apple 추가.
3. 빌드.

## 3. 사용 가능 여부 체크

```ts
import appleAuth from '@invertase/react-native-apple-authentication';

if (!appleAuth.isSupported) {
  // iOS 13 미만 또는 Android
  return null;
}
```

→ **iOS 13+** 필요.

## 4. 로그인 요청

```ts
import appleAuth, {
  AppleAuthRequestOperation,
  AppleAuthRequestScope,
  AppleAuthCredentialState,
} from '@invertase/react-native-apple-authentication';

async function signInWithApple() {
  try {
    const response = await appleAuth.performRequest({
      requestedOperation: AppleAuthRequestOperation.LOGIN,
      requestedScopes: [AppleAuthRequestScope.EMAIL, AppleAuthRequestScope.FULL_NAME],
    });

    const {
      user,                  // Apple user ID
      email,                 // 첫 로그인만! 그 후엔 null
      fullName,              // 첫 로그인만!
      identityToken,         // JWT — BE 에 전달
      authorizationCode,     // 짧은 lifetime
      realUserStatus,        // 0=unsupported / 1=unknown / 2=likelyReal
    } = response;

    if (!identityToken) throw new Error('no token');

    // BE 에 검증
    const { data } = await api.post('/auth/apple', {
      idToken: identityToken,
      authorizationCode,
      email,                  // 첫 가입 시만 의미 있음
      fullName,
    });

    // 저장 + 이동
    await tokenStorage.setAccess(data.accessToken);
    await tokenStorage.setRefresh(data.refreshToken);
    navigation.reset({ index: 0, routes: [{ name: 'MainTab' }] });
  } catch (e) {
    if (e.code === appleAuth.Error.CANCELED) {
      // 사용자가 취소
    } else {
      console.error(e);
    }
  }
}
```

## 5. BE 가 받은 idToken 처리

BE 가 해야 할 일:
1. `identityToken` (JWT) 의 signature 검증 — Apple 의 public key.
2. `aud` (audience) = 우리 client_id 확인.
3. `iss` = `https://appleid.apple.com` 확인.
4. `exp` 만료 확인.
5. 첫 가입이면 email / fullName 저장 (Apple 은 이 후 응답 X).

## 6. 첫 로그인 vs 재로그인

```
첫 로그인 (가입):
  response = { user, email, fullName, identityToken, ... }
  → 모든 필드 채움

재로그인 (두 번째 부터):
  response = { user, identityToken, ... }    ← email/fullName null
  → BE 가 user ID 로 기존 user 찾아야
```

→ **BE 가 첫 응답의 email/fullName 을 반드시 저장**.

## 7. credential state 확인 — 자동 로그아웃 감지

```ts
useEffect(() => {
  (async () => {
    const user = await AsyncStorage.getItem('apple_user_id');
    if (!user) return;

    const state = await appleAuth.getCredentialStateForUser(user);
    if (state === AppleAuthCredentialState.REVOKED) {
      // 사용자가 설정 → Apple ID → Apple 로 로그인 → 우리 앱 revoke
      await logout();
    }
  })();
}, []);
```

→ 사용자가 시스템 설정에서 우리 앱의 Apple 권한 취소 가능. 감지 + logout.

## 8. 회원 탈퇴 — Apple 의 revoke (server-side)

```
1. FE: BE 에 탈퇴 요청
2. BE: Apple revoke API 호출
       POST https://appleid.apple.com/auth/revoke
       client_id, client_secret, token, token_type_hint
3. BE: 우리 DB 의 user 삭제
```

→ Apple 의 가이드라인 (2022+): **앱 안에서 회원 탈퇴 + Apple ID 의 권한도 revoke 필수**. 어김 시 App Store 리젝.

## 9. UI — Apple 의 공식 버튼

Apple HIG 에 따라 디자인 가이드 엄격:
- 흰 배경 + 검은 텍스트, 검은 배경 + 흰 텍스트, 검은 outline.
- "Sign in with Apple" 글자 (다국어 자동).
- corner-radius / 폰트 / 크기 규정.

라이브러리 제공:
```tsx
import { AppleButton } from '@invertase/react-native-apple-authentication';

<AppleButton
  buttonStyle={AppleButton.Style.BLACK}
  buttonType={AppleButton.Type.SIGN_IN}
  style={{ width: '100%', height: 48 }}
  onPress={signInWithApple}
/>
```

→ Apple 가이드 자동 준수. 직접 디자인 시 위반 위험.

## 10. Android 의 Apple Sign In

Apple SDK 는 **iOS 전용**. Android 에서 Apple 로그인이 필요하면:
- Web view (`react-native-inappbrowser-reborn`) + Apple OAuth redirect.
- Or `react-native-apple-authentication-android` 같은 third-party.
- 또는 그냥 Android 에서 Apple 버튼 안 보이게.

```tsx
{Platform.OS === 'ios' && <AppleButton ... />}
```

→ job-answer-app-rn 도 iOS-only.

## 11. job-answer-app-rn 의 패턴

```tsx
// components/SocialButton/Apple.tsx
import { Platform } from 'react-native';
import appleAuth, { AppleAuthRequestOperation, AppleAuthRequestScope } from '@invertase/react-native-apple-authentication';

export const AppleLoginButton = () => {
  if (Platform.OS !== 'ios' || !appleAuth.isSupported) return null;

  const handle = async () => {
    try {
      const res = await appleAuth.performRequest({
        requestedOperation: AppleAuthRequestOperation.LOGIN,
        requestedScopes: [AppleAuthRequestScope.EMAIL, AppleAuthRequestScope.FULL_NAME],
      });
      // BE 전송
      const { accessToken, refreshToken } = await authApi.appleLogin({
        idToken: res.identityToken!,
        authorizationCode: res.authorizationCode!,
        email: res.email,
        fullName: res.fullName,
      });
      await tokenStorage.setAccess(accessToken);
      await tokenStorage.setRefresh(refreshToken);
      navigation.reset({ index: 0, routes: [{ name: 'MainTab' }] });
    } catch (e) {
      if (e.code !== appleAuth.Error.CANCELED) Alert.alert('로그인 실패');
    }
  };

  return <AppleButton onPress={handle} ... />;
};
```

## 12. 함정

1. **identityToken 없음** — scope 누락 또는 user 가 권한 거부.
2. **email null** — 첫 가입만 응답. BE 저장 누락 시 영영 모름.
3. **`appleAuth.isSupported` 안 체크** — iOS 12 / Android 에서 throw.
4. **revoke 처리 누락** — App Store 리젝.
5. **Apple Developer 의 Capability 누락** — `code=1000 unknown error`.
6. **realUserStatus = 0** — 시뮬레이터에선 unknown. 실기기에서 확인.
7. **자동 로그인 + credential REVOKED** — 옛 token 으로 fail 무한 retry. credentialState 체크.

## 13. 외부 자료

- [@invertase/react-native-apple-authentication](https://github.com/invertase/react-native-apple-authentication)
- [Apple — Sign in with Apple](https://developer.apple.com/sign-in-with-apple/)
- [App Store Review Guidelines 4.8](https://developer.apple.com/app-store/review/guidelines/#sign-in-with-apple)

## 14. 관련

- [[auth]]
- [[google-signin]]
- [[kakao-login]]
