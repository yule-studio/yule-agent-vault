---
title: "OAuth 소셜 로그인 — Google / Kakao / Apple"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, auth, oauth, kakao, google, apple]
---

# OAuth 소셜 로그인

**[[auth|↑ Auth Hub]]**

> Google / Kakao / Apple 등 외부 인증으로 로그인.

## 1. OAuth 2.0 의 큰 그림

```
1. 사용자: "카카오로 로그인" 클릭
       ↓
2. FE → Kakao 인증 페이지로 redirect
   https://kauth.kakao.com/oauth/authorize?client_id=...&redirect_uri=...&response_type=code
       ↓
3. 사용자: 카카오 계정 로그인 + 권한 동의
       ↓
4. Kakao → 우리 redirect_uri 로 redirect (code 포함)
   https://myapp.com/auth/kakao/callback?code=ABC123
       ↓
5. FE → BE: code 전달
       ↓
6. BE → Kakao: code + client_secret → access token + user info
       ↓
7. BE → FE: 우리의 JWT (또는 session)
       ↓
8. 로그인 완료
```

→ 핵심: **secret 은 BE 만 알고, FE 는 code 만 전달**.

## 2. 사전 준비 — 각 provider 콘솔 설정

- [Kakao developers](https://developers.kakao.com/) 앱 생성 → redirect URI 등록.
- [Google Cloud Console](https://console.cloud.google.com/) → OAuth client ID 생성.
- [Apple Developer](https://developer.apple.com/) → Sign in with Apple 등록.

→ `redirect_uri` 정확히 일치해야 (trailing slash, http/https, port 모두).

## 3. 환경 변수

```env
VITE_KAKAO_CLIENT_ID=...
VITE_GOOGLE_CLIENT_ID=...
VITE_REDIRECT_URI=https://myapp.com/auth/callback
```

→ client_id 는 공개 OK (secret 만 BE).

## 4. 시작 버튼 — provider 로 redirect

```tsx
// LoginPage.tsx

const KAKAO_AUTH = `https://kauth.kakao.com/oauth/authorize?client_id=${import.meta.env.VITE_KAKAO_CLIENT_ID}&redirect_uri=${import.meta.env.VITE_REDIRECT_URI}/kakao&response_type=code`;

const GOOGLE_AUTH = `https://accounts.google.com/o/oauth2/v2/auth?client_id=${import.meta.env.VITE_GOOGLE_CLIENT_ID}&redirect_uri=${import.meta.env.VITE_REDIRECT_URI}/google&response_type=code&scope=email%20profile`;

<a href={KAKAO_AUTH}>카카오로 로그인</a>
<a href={GOOGLE_AUTH}>구글로 로그인</a>
```

또는 `window.location.href = KAKAO_AUTH`.

## 5. 콜백 페이지 — code 받아 BE 에 전달

```tsx
// /auth/kakao/callback 라우트
import { useEffect } from 'react';
import { useNavigate, useSearchParams } from 'react-router-dom';

function KakaoCallback() {
  const navigate = useNavigate();
  const [params] = useSearchParams();
  const code = params.get('code');

  useEffect(() => {
    if (!code) {
      navigate('/login?error=missing_code');
      return;
    }

    api.post('/auth/kakao/callback', { code })
      .then(({ data }) => {
        tokenStorage.setAccess(data.accessToken);
        tokenStorage.setRefresh(data.refreshToken);
        navigate('/');
      })
      .catch(() => navigate('/login?error=oauth_failed'));
  }, [code]);

  return <Spinner />;
}

// App.tsx
<Routes>
  <Route path="/auth/kakao/callback" element={<KakaoCallback />} />
  <Route path="/auth/google/callback" element={<GoogleCallback />} />
</Routes>
```

## 6. Google — `@react-oauth/google` 라이브러리

```bash
yarn add @react-oauth/google
```

```tsx
import { GoogleOAuthProvider, GoogleLogin } from '@react-oauth/google';

<GoogleOAuthProvider clientId={import.meta.env.VITE_GOOGLE_CLIENT_ID}>
  <GoogleLogin
    onSuccess={(credentialResponse) => {
      // credentialResponse.credential = ID token (JWT)
      api.post('/auth/google', { credential: credentialResponse.credential })
        .then(r => {
          tokenStorage.setAccess(r.data.accessToken);
          navigate('/');
        });
    }}
    onError={() => console.log('Login Failed')}
  />
</GoogleOAuthProvider>
```

→ ID token 방식이 더 간단 (FE 가 JWT 받음 → BE 검증).

## 7. Apple — Sign in with Apple

```html
<!-- index.html -->
<script
  src="https://appleid.cdn-apple.com/appleauth/static/jsapi/appleid/1/en_US/appleid.auth.js"
  async
></script>
```

```tsx
useEffect(() => {
  (window as any).AppleID.auth.init({
    clientId: import.meta.env.VITE_APPLE_CLIENT_ID,
    scope: 'email name',
    redirectURI: `${window.location.origin}/auth/apple/callback`,
    state: 'random-state',
    usePopup: true,
  });
}, []);

const handleAppleLogin = async () => {
  const data = await (window as any).AppleID.auth.signIn();
  // data.authorization.id_token = Apple ID token
  await api.post('/auth/apple', { idToken: data.authorization.id_token });
};

<button onClick={handleAppleLogin}>Apple 로그인</button>
```

→ Apple 은 첫 로그인 시에만 이름 / 이메일 응답. BE 가 저장 필수.

## 8. Kakao — JS SDK (선택적)

```html
<!-- index.html -->
<script src="https://t1.kakaocdn.net/kakao_js_sdk/2.6.0/kakao.min.js" />
```

```ts
(window as any).Kakao.init(import.meta.env.VITE_KAKAO_JS_KEY);

(window as any).Kakao.Auth.login({
  success: (response) => {
    api.post('/auth/kakao', { accessToken: response.access_token });
  },
});
```

→ SDK 없이 redirect 방식이 더 간단.

## 9. CSRF 방어 — state 파라미터

```ts
const state = crypto.randomUUID();
sessionStorage.setItem('oauth_state', state);

window.location.href = `${KAKAO_AUTH}&state=${state}`;
```

```ts
// 콜백
const returnedState = params.get('state');
const stored = sessionStorage.getItem('oauth_state');
if (returnedState !== stored) {
  throw new Error('CSRF detected');
}
```

→ 콜백 요청이 진짜 우리가 보낸 것인지 검증.

## 10. PKCE (Proof Key for Code Exchange) — SPA 권장

```ts
// 1. code_verifier 생성 (random)
const verifier = generateRandomString(64);
const challenge = base64url(sha256(verifier));
sessionStorage.setItem('pkce_verifier', verifier);

// 2. redirect 시 challenge 전달
location.href = `...&code_challenge=${challenge}&code_challenge_method=S256`;

// 3. 콜백에서 token 교환 시 verifier 전달
api.post('/auth/token', { code, codeVerifier: sessionStorage.getItem('pkce_verifier') });
```

→ SPA 에서 client_secret 못 쓰는 환경의 대안. 최신 OAuth 권장.

## 11. answer-fe 의 흔한 OAuth 패턴

```ts
// lib/oauth/kakao.ts
export const kakaoOAuth = {
  getAuthUrl: () => `https://kauth.kakao.com/oauth/authorize?...`,
  handleCallback: async (code: string) => {
    const { data } = await api.post('/auth/kakao/callback', { code });
    return data;
  },
};

// pages/auth/KakaoCallback.tsx
function KakaoCallback() {
  const [params] = useSearchParams();
  const navigate = useNavigate();

  useEffect(() => {
    const code = params.get('code');
    if (!code) return navigate('/login');

    kakaoOAuth.handleCallback(code)
      .then(({ accessToken, isNewUser, user }) => {
        tokenStorage.setAccess(accessToken);
        navigate(isNewUser ? '/onboarding' : '/');
      })
      .catch(() => navigate('/login'));
  }, []);

  return <Spinner />;
}
```

## 12. 첫 로그인 vs 재로그인 분기

```ts
// BE 응답
{
  accessToken: '...',
  refreshToken: '...',
  user: { id, email, name },
  isNewUser: true,            // 처음 로그인
}

// FE
if (isNewUser) navigate('/onboarding');
else navigate('/');
```

## 13. 함정

1. **redirect_uri 불일치** — 콘솔의 등록 URL 과 실제 URL 1 글자도 안 맞으면 실패.
2. **code 1 회용** — 콜백 useEffect 가 dev mode 에서 2 번 실행 → 두 번째 호출은 invalid_grant. Strict Mode + `useRef` guard 또는 BE 에서 redundant check.
3. **state 미검증** — CSRF 가능.
4. **token 의 FE 노출** — provider 의 access token 받지 말고 BE 의 JWT 만.
5. **scope 과다** — 꼭 필요한 정보만. 사용자 동의율 떨어짐.
6. **Apple 의 첫 응답에만 이름/이메일** — BE 가 저장 필수. 두 번째 로그인부터는 못 받음.
7. **`useEffect` 의 의존성 누락** — code 가 바뀌어도 안 처리.

## 14. 외부 자료

- [Kakao OAuth](https://developers.kakao.com/docs/latest/ko/kakaologin/rest-api)
- [Google OAuth 2.0](https://developers.google.com/identity/protocols/oauth2)
- [@react-oauth/google](https://www.npmjs.com/package/@react-oauth/google)
- [Apple Sign In](https://developer.apple.com/sign-in-with-apple/)

## 15. 관련

- [[auth]]
- [[login-jwt]]
- [[../http/interceptors]]
