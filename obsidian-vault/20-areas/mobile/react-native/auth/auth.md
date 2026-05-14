---
title: "Authentication in RN — 소셜 로그인 (Apple / Google / Kakao)"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, auth, social-login, hub]
---

# Authentication in RN

**[[../react-native|↑ RN Hub]]**

> 모바일은 web 과 달리 **각 provider 의 native SDK** 사용. job-answer-app-rn 의 핵심.

## 1. RN 의 소셜 로그인 — Web 과 차이

| | Web | RN |
| --- | --- | --- |
| 흐름 | redirect → callback URL | native SDK 호출 |
| UX | 브라우저 새 탭 / 페이지 이동 | 앱 안 native modal |
| Apple | OAuth | **Sign in with Apple (iOS native)** |
| Google | OAuth redirect | GoogleSignIn native SDK |
| Kakao | OAuth redirect | Kakao native SDK (앱 / 카톡 앱 연동) |

→ 사용자 경험이 native = 마찰 적음 + 자동 로그인 가능.

## 2. job-answer-app-rn 의 lib (v5+)

```json
"@invertase/react-native-apple-authentication": "^2.5.0",
"@react-native-google-signin/google-signin": "^16.0.0",
"@react-native-seoul/kakao-login": "^5.4.2"
```

→ 각각 native module. iOS / Android 의 pod / gradle 자동 link.

## 3. 통합 흐름 — 3 사 공통 패턴

```
1. 사용자: "Apple/Google/Kakao 로그인" 버튼 클릭
       ↓
2. FE: native SDK 호출 (provider 가 native UI 표시)
       ↓
3. 사용자: 인증 + 권한 동의
       ↓
4. FE: provider 의 id_token / access_token 받음
       ↓
5. FE → BE: token 전달 (`POST /auth/{provider}`)
       ↓
6. BE: provider 에 token 검증 + 우리 user 찾기 / 생성
       ↓
7. BE → FE: 우리 JWT (access + refresh)
       ↓
8. FE: AsyncStorage 저장 + navigation reset → MainTab
```

## 4. 학습 우선순위

1. **[[apple-signin]]** — `@invertase/react-native-apple-authentication`.
2. **[[google-signin]]** — `@react-native-google-signin/google-signin`.
3. **[[kakao-login]]** — `@react-native-seoul/kakao-login`.
4. **token 저장** — [[../state/async-storage]].
5. **axios interceptor + 401 refresh** — [[../networking/axios-fetch]].
6. **인증 흐름 / navigator 전환** — [[../navigation/navigation]].

## 5. SocialLoginSection 컴포넌트 패턴

```tsx
// components/SocialLoginSection.tsx (job-answer-app-rn 패턴)
import { Platform } from 'react-native';

export function SocialLoginSection() {
  return (
    <View>
      {Platform.OS === 'ios' && <AppleLoginButton onSuccess={onAppleSuccess} />}
      <KakaoLoginButton onSuccess={onKakaoSuccess} />
      <GoogleLoginButton onSuccess={onGoogleSuccess} />
    </View>
  );
}

const onAppleSuccess = async (token: string) => {
  const { data } = await api.post('/auth/apple', { idToken: token });
  await tokenStorage.setAccess(data.accessToken);
  await tokenStorage.setRefresh(data.refreshToken);
  navigation.reset({ index: 0, routes: [{ name: 'MainTab' }] });
};
// onKakaoSuccess, onGoogleSuccess 도 유사
```

## 6. 인증 흐름 — 앱 시작 시

```tsx
// App.tsx
function App() {
  const { user, isLoading } = useAuth();

  if (isLoading) return <Splash />;

  return (
    <NavigationContainer>
      {user ? <MainStack /> : <AuthStack />}
    </NavigationContainer>
  );
}

// hooks/useAuth.ts
export function useAuth() {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    (async () => {
      const token = await tokenStorage.getAccess();
      if (!token) { setIsLoading(false); return; }
      try {
        const { data } = await api.get('/users/me');
        setUser(data);
      } catch {
        await tokenStorage.clear();
      } finally {
        setIsLoading(false);
      }
    })();
  }, []);

  return { user, isLoading };
}
```

## 7. token 저장

```ts
// lib/auth/tokenStorage.ts
import AsyncStorage from '@react-native-async-storage/async-storage';

export const tokenStorage = {
  getAccess: () => AsyncStorage.getItem('access_token'),
  setAccess: (t: string) => AsyncStorage.setItem('access_token', t),
  getRefresh: () => AsyncStorage.getItem('refresh_token'),
  setRefresh: (t: string) => AsyncStorage.setItem('refresh_token', t),
  clear: () => AsyncStorage.multiRemove(['access_token', 'refresh_token']),
};
```

→ 보안 강화 → react-native-keychain. 자세히 [[../state/async-storage]].

## 8. 로그아웃 (3 가지 단계)

```ts
async function logout() {
  try {
    // 1. BE 에 logout 알림 (refresh 무효화)
    await api.post('/auth/logout');
  } catch {}

  // 2. provider 의 native 로그아웃
  try { await GoogleSignin.signOut(); } catch {}
  try { await KakaoLogins.logout(); } catch {}
  // Apple 은 native 로그아웃 없음

  // 3. 로컬 정리
  await tokenStorage.clear();
  qc.clear();                              // react-query cache
  navigation.reset({ index: 0, routes: [{ name: 'Signin' }] });
}
```

## 9. 회원 탈퇴 — 각 provider 의 revoke

```ts
async function deleteAccount() {
  // 1. provider 의 revoke
  await GoogleSignin.revokeAccess();
  await KakaoLogins.unlink();
  // Apple — Apple ID 의 revoke endpoint 호출 (BE)

  // 2. BE 에 탈퇴 요청
  await api.delete('/users/me');

  // 3. 로컬 정리
  await tokenStorage.clear();
}
```

→ Apple 의 revoke 는 server-to-server. BE 가 처리.

## 10. PrivateScreen / route guard

```tsx
// 컴포넌트 안에서 한 줄
const { user } = useAuth();
useEffect(() => {
  if (!user) navigation.replace('Signin');
}, [user]);
```

→ 또는 별도 stack 분리 (위 §6).

## 11. 보안 권장 사항

- **token 의 평문 저장** — XSS 거의 없지만 rooted device 위험. keychain 권장.
- **id_token / access_token 의 BE 검증** — FE 가 받은 token 을 BE 가 provider 에 재검증.
- **HTTPS only** — 모든 BE 통신 HTTPS.
- **refresh token rotation** — BE 가 refresh 마다 새 token 발급.

## 12. 함정

1. **redirect URI 와 deep link 혼동** — 모바일 native SDK 는 deep link 불필요.
2. **`Platform.OS === 'ios'` 누락** — Apple Sign In 을 Android 에서 시도 → 에러.
3. **첫 로그인 시에만 email/name 응답** — Apple. BE 가 저장 필수.
4. **provider 의 idToken vs accessToken** — Apple = idToken, Google = idToken+accessToken, Kakao = accessToken+refreshToken.
5. **dev 환경의 SHA-1 mismatch** — Android Google SignIn 의 흔한 에러.
6. **logout 후 provider 도 logout 안 함** — 다음 로그인 시 자동 같은 계정 (불편).

## 13. 다음 단계

- [[apple-signin]]
- [[google-signin]]
- [[kakao-login]]

## 14. 관련

- [[../react-native]]
- [[../state/async-storage]]
- [[../networking/axios-fetch]]
- [[../navigation/navigation]]
- [[../../../frontend/react/auth/auth|web auth]]
