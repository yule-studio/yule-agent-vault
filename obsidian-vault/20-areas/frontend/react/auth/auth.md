---
title: "Authentication — JWT, OAuth, route 가드"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, auth, jwt, oauth, hub]
---

# Authentication

**[[../react|↑ React Hub]]**

> 로그인 / 인증 / 권한. **token 저장 / refresh / route 보호 / OAuth** 의 4 가지가 핵심.

## 1. 인증의 전체 흐름

```
1. 로그인 (email/pw 또는 OAuth)
       ↓
2. 서버가 access token + refresh token 발급
       ↓
3. FE 가 token 저장 (cookie / localStorage)
       ↓
4. 이후 모든 요청 → Authorization: Bearer <token>
       ↓
5. access token 만료 → refresh token 으로 재발급
       ↓
6. refresh token 도 만료 → 다시 로그인
```

## 2. token 저장 — 어디에?

| | 장점 | 단점 |
| --- | --- | --- |
| **localStorage** | 쉬움, XHR 자동 attach | XSS 시 노출 |
| **sessionStorage** | tab 닫으면 만료 | XSS 시 노출 |
| **httpOnly cookie** | XSS 안전 | CSRF 대비 필요, 설정 복잡 |
| **In-memory** | 가장 안전 | 새로고침 시 사라짐 |

→ 보안 최우선 = **httpOnly + Secure + SameSite cookie**.
→ 개발 빠르기 = **localStorage** (XSS 방어 강화 동반).

`masterway-dev/*-fe` 는 보통 **localStorage** 사용.

## 3. 로그인 (이메일/비밀번호) — [[login-jwt]]

```tsx
const { mutate: login } = useMutation({
  mutationFn: (data) => api.post('/auth/login', data).then(r => r.data),
  onSuccess: ({ accessToken, refreshToken, user }) => {
    localStorage.setItem('access_token', accessToken);
    localStorage.setItem('refresh_token', refreshToken);
    setUser(user);
    navigate('/');
  },
});
```

자세히 [[login-jwt]].

## 4. OAuth 소셜 로그인 — [[oauth-social]]

```tsx
<a href="https://kauth.kakao.com/oauth/authorize?client_id=...">
  카카오 로그인
</a>

// 콜백 페이지
const code = new URLSearchParams(location.search).get('code');
await api.post('/auth/kakao/callback', { code });
```

자세히 [[oauth-social]].

## 5. token 자동 첨부 — axios interceptor

```ts
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('access_token');
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});
```

→ 자세히 [[../http/interceptors]].

## 6. refresh token — 401 자동 재발급

```ts
api.interceptors.response.use(
  r => r,
  async (error) => {
    if (error.response?.status === 401 && !error.config._retry) {
      error.config._retry = true;
      const newToken = await refreshAccessToken();
      error.config.headers.Authorization = `Bearer ${newToken}`;
      return api(error.config);
    }
    return Promise.reject(error);
  }
);
```

→ 자세히 [[../http/interceptors]].

## 7. 인증 상태 — 어디 저장?

```tsx
// 방법 1 — zustand store
const useAuth = create<AuthState>((set) => ({
  user: null,
  token: localStorage.getItem('access_token'),
  setUser: (user) => set({ user }),
  logout: () => {
    localStorage.removeItem('access_token');
    localStorage.removeItem('refresh_token');
    set({ user: null, token: null });
  },
}));

// 방법 2 — react-query (서버 상태처럼)
const useMe = () => useQuery({
  queryKey: ['me'],
  queryFn: () => api.get('/users/me').then(r => r.data),
  enabled: !!localStorage.getItem('access_token'),
  staleTime: Infinity,
});
```

→ token = client 상태 (zustand / jotai), user = 서버 상태 (react-query). 권장.

## 8. Route 가드 — PrivateRoute

```tsx
// PrivateRoute.tsx
import { Navigate, Outlet, useLocation } from 'react-router-dom';

function PrivateRoute() {
  const { data: me, isLoading } = useMe();
  const location = useLocation();

  if (isLoading) return <Spinner />;
  if (!me) return <Navigate to="/login" state={{ from: location }} replace />;

  return <Outlet />;
}

// App.tsx
<Routes>
  <Route element={<PrivateRoute />}>
    <Route path="/mypage" element={<MyPage />} />
    <Route path="/dashboard" element={<Dashboard />} />
  </Route>
  <Route path="/login" element={<Login />} />
</Routes>
```

```tsx
// Login.tsx — 로그인 후 원래 가려던 페이지로
const location = useLocation();
const navigate = useNavigate();
const from = location.state?.from?.pathname ?? '/';
await login(...);
navigate(from, { replace: true });
```

## 9. role / 권한 (RBAC)

```tsx
function RoleRoute({ allow }: { allow: Role[] }) {
  const { data: me } = useMe();
  if (!me) return <Navigate to="/login" />;
  if (!allow.includes(me.role)) return <Navigate to="/forbidden" />;
  return <Outlet />;
}

<Routes>
  <Route element={<RoleRoute allow={['ADMIN']} />}>
    <Route path="/admin" element={<AdminPanel />} />
  </Route>
</Routes>
```

→ 더 큰 시스템은 권한 매트릭스 (resource × action).

## 10. 로그아웃

```ts
const logout = async () => {
  try {
    await api.post('/auth/logout');   // 서버에 refresh 무효화
  } catch {}
  localStorage.removeItem('access_token');
  localStorage.removeItem('refresh_token');
  qc.clear();                         // react-query 캐시 비움
  navigate('/login');
};
```

## 11. 자동 로그인 / "로그인 유지"

```ts
// 로그인 시 체크박스
if (rememberMe) {
  localStorage.setItem('refresh_token', refreshToken);
} else {
  sessionStorage.setItem('refresh_token', refreshToken);
}
```

→ tab 닫으면 만료시킬지 / 영구 유지할지.

## 12. 첫 진입 시 me fetch

```tsx
// App 마운트 시 자동 fetch
function App() {
  const { data: me, isLoading } = useMe();   // token 있으면 자동
  if (isLoading) return <Splash />;
  return <Routes>...</Routes>;
}
```

## 13. 보안 고려

| | 대응 |
| --- | --- |
| **XSS** | dompurify, CSP, httpOnly cookie |
| **CSRF** | SameSite=strict, CSRF token |
| **token leak** | 짧은 access token + refresh |
| **brute force** | reCAPTCHA, rate limit (백엔드) |
| **password 노출** | HTTPS only, 평문 저장 금지 |

## 14. 학습 우선순위

1. **[[login-jwt]]** — 이메일/비밀번호 로그인 + JWT.
2. **[[oauth-social]]** — Google / Kakao / Apple OAuth.
3. **PrivateRoute / role guard**.
4. **interceptor refresh** ([[../http/interceptors]]).
5. **logout / token cleanup**.

## 15. 함정

1. **localStorage 의 XSS** — DOMPurify 로 HTML sanitize. 사용자 입력 innerHTML ❌.
2. **token 의 JS 노출** — XSS 한번 = token 노출. httpOnly cookie 가 가장 안전.
3. **logout 시 react-query cache 안 비움** — 다음 사용자 로그인 시 옛 데이터 표시. `qc.clear()` 필수.
4. **refresh token 의 무한 loop** — refresh 도 401 → 또 refresh → ... `_retry` 플래그.
5. **OAuth redirect URL 의 trailing slash** — 등록 URL 과 정확히 일치해야.
6. **token expiration check** — 시계 어긋남 가능. 5 ~ 10 초 buffer.

## 16. 외부 자료

- [JWT.io](https://jwt.io/) — JWT decode / debug.
- [OWASP Auth Cheatsheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OAuth 2.0 RFC](https://datatracker.ietf.org/doc/html/rfc6749)

## 17. 관련

- [[login-jwt]]
- [[oauth-social]]
- [[../http/interceptors]]
- [[../routing/react-router-v6]]
