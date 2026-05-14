---
title: "Login + JWT — 이메일/비밀번호 + access/refresh token"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, auth, jwt, login]
---

# Login + JWT

**[[auth|↑ Auth Hub]]**

> 이메일/비밀번호 로그인 + JWT (access + refresh) 의 표준 플로우.

## 1. 전체 그림

```
사용자: email + password 입력
   ↓
FE: POST /auth/login { email, password }
   ↓
BE: 검증 → { accessToken (15분), refreshToken (14일), user }
   ↓
FE: token 저장 + Authorization 헤더에 자동 첨부
   ↓
access 만료 → 401 → refresh 로 재발급 → 원 요청 재시도
   ↓
refresh 도 만료 → logout
```

## 2. JWT 의 구조

```
header.payload.signature
```

```json
// payload (base64 decode)
{
  "sub": "user-id-123",
  "email": "yc@example.com",
  "role": "USER",
  "exp": 1715000000,           // 만료 timestamp
  "iat": 1714900000
}
```

→ FE 는 payload 만 읽기 가능 (`jwt-decode` 라이브러리). signature 는 서버만 검증.

```ts
import { jwtDecode } from 'jwt-decode';

const payload = jwtDecode<{ sub: string; exp: number }>(token);
const isExpired = payload.exp * 1000 < Date.now();
```

## 3. Login form — react-hook-form + zod

```tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const schema = z.object({
  email: z.string().email('이메일 형식'),
  password: z.string().min(8, '8자 이상'),
});

type FormData = z.infer<typeof schema>;

function LoginForm() {
  const { register, handleSubmit, formState: { errors, isSubmitting } } = useForm<FormData>({
    resolver: zodResolver(schema),
  });

  const navigate = useNavigate();
  const { setUser, setToken } = useAuth();

  const onSubmit = async (data: FormData) => {
    try {
      const { accessToken, refreshToken, user } = await api.post('/auth/login', data).then(r => r.data);
      localStorage.setItem('access_token', accessToken);
      localStorage.setItem('refresh_token', refreshToken);
      setToken(accessToken);
      setUser(user);
      navigate('/');
    } catch (e) {
      // 에러 message
    }
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input type="email" {...register('email')} />
      {errors.email && <p>{errors.email.message}</p>}
      <input type="password" {...register('password')} />
      {errors.password && <p>{errors.password.message}</p>}
      <button type="submit" disabled={isSubmitting}>로그인</button>
    </form>
  );
}
```

## 4. Token 저장소

```ts
// lib/tokenStorage.ts
export const tokenStorage = {
  getAccess: () => localStorage.getItem('access_token'),
  setAccess: (t: string) => localStorage.setItem('access_token', t),
  getRefresh: () => localStorage.getItem('refresh_token'),
  setRefresh: (t: string) => localStorage.setItem('refresh_token', t),
  clear: () => {
    localStorage.removeItem('access_token');
    localStorage.removeItem('refresh_token');
  },
};
```

→ 1 곳에서 관리. 추후 httpOnly cookie 로 교체 시 이 파일만 변경.

## 5. axios interceptor — token 첨부

```ts
api.interceptors.request.use((config) => {
  const token = tokenStorage.getAccess();
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});
```

## 6. Refresh token — 401 자동 재발급

```ts
let isRefreshing = false;
let waitQueue: Array<(t: string) => void> = [];

api.interceptors.response.use(
  r => r,
  async (error) => {
    const original = error.config;
    if (error.response?.status !== 401 || original._retry) {
      return Promise.reject(error);
    }

    if (isRefreshing) {
      return new Promise((resolve) => {
        waitQueue.push((token) => {
          original.headers.Authorization = `Bearer ${token}`;
          resolve(api(original));
        });
      });
    }

    original._retry = true;
    isRefreshing = true;

    try {
      const refresh = tokenStorage.getRefresh();
      if (!refresh) throw new Error('no refresh');

      const { data } = await axios.post('/auth/refresh', { refreshToken: refresh });
      tokenStorage.setAccess(data.accessToken);

      waitQueue.forEach(cb => cb(data.accessToken));
      waitQueue = [];

      original.headers.Authorization = `Bearer ${data.accessToken}`;
      return api(original);
    } catch (e) {
      tokenStorage.clear();
      window.location.href = '/login';
      return Promise.reject(e);
    } finally {
      isRefreshing = false;
    }
  }
);
```

→ 핵심: queue 패턴. 동시 401 들이 1 번 refresh 후 모두 재시도.

## 7. 현재 사용자 — useMe

```tsx
import { useQuery } from '@tanstack/react-query';

export const useMe = () => {
  return useQuery({
    queryKey: ['me'],
    queryFn: () => api.get('/users/me').then(r => r.data),
    enabled: !!tokenStorage.getAccess(),
    staleTime: 5 * 60 * 1000,
    retry: false,
  });
};

// 사용
function Header() {
  const { data: me } = useMe();
  return <div>{me ? `${me.name} 님` : <Link to="/login">로그인</Link>}</div>;
}
```

## 8. Logout

```ts
const logout = async () => {
  try {
    await api.post('/auth/logout');     // refresh 무효화
  } catch {}
  tokenStorage.clear();
  qc.clear();                            // react-query 캐시 비움
  navigate('/login');
};
```

## 9. 자동 로그인 (앱 시작 시)

```tsx
function App() {
  const { isLoading } = useMe();           // token 있으면 자동 fetch
  if (isLoading) return <Splash />;
  return <Router>...</Router>;
}
```

→ token 있으면 me fetch → 성공 시 로그인 상태, 실패 시 자동 logout.

## 10. JWT 의 만료 미리 감지

```ts
// 토큰 만료 1 분 전 미리 refresh
const scheduleRefresh = (token: string) => {
  const { exp } = jwtDecode<{ exp: number }>(token);
  const msUntil = exp * 1000 - Date.now() - 60_000;
  if (msUntil > 0) {
    setTimeout(() => refreshAccessToken(), msUntil);
  }
};
```

→ background 갱신. 401 의존 X.

## 11. answer-fe / job-answer-fe 패턴

```ts
// lib/auth/api.ts
export const authApi = {
  login: (body: LoginDto) => api.post<LoginResponse>('/auth/login', body).then(r => r.data),
  refresh: (refresh: string) => api.post<{ accessToken: string }>('/auth/refresh', { refreshToken: refresh }).then(r => r.data),
  logout: () => api.post('/auth/logout'),
  me: () => api.get<User>('/users/me').then(r => r.data),
};

// hooks/useLogin.ts
export const useLogin = () => {
  const navigate = useNavigate();
  return useMutation({
    mutationFn: authApi.login,
    onSuccess: ({ accessToken, refreshToken, user }) => {
      tokenStorage.setAccess(accessToken);
      tokenStorage.setRefresh(refreshToken);
      navigate('/');
    },
    onError: (e) => {
      message.error('로그인 실패');
    },
  });
};
```

## 12. 보안 강화 — httpOnly cookie 로 전환

```
1. 로그인 응답: Set-Cookie: refresh_token=...; HttpOnly; Secure; SameSite=Strict
2. FE 는 refresh token 직접 읽지 못함.
3. 401 시 자동으로 cookie 가 /auth/refresh 에 전송 → 새 access token.
4. access token 만 메모리 or localStorage.

→ XSS 로부터 refresh 안전. CSRF 는 SameSite + CSRF token 으로.
```

`axios.create({ withCredentials: true })` 필수.

## 13. 함정

1. **JWT 의 password 저장** — 절대 안 됨. payload 는 누구나 decode 가능.
2. **token 의 localStorage 와 XSS** — DOMPurify, CSP, innerHTML 금지.
3. **refresh 의 무한 loop** — `_retry` 플래그로 방지.
4. **logout 시 cache 안 비움** — `qc.clear()`.
5. **token 의 시계 어긋남** — 5 ~ 10 초 buffer.
6. **로그인 form 의 brute force** — reCAPTCHA 또는 백엔드 rate limit.
7. **HTTPS 누락** — 로그인 form 은 반드시 HTTPS.

## 14. 외부 자료

- [JWT.io](https://jwt.io/)
- [jwt-decode](https://www.npmjs.com/package/jwt-decode)
- [OWASP JWT cheatsheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)

## 15. 관련

- [[auth]]
- [[oauth-social]]
- [[../http/interceptors]]
- [[../forms/react-hook-form]]
