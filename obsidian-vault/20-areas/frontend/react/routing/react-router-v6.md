---
title: "react-router-dom v6 — 실전 사용"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, react-router, routing, spa]
---

# react-router-dom v6

**[[routing|↑ Routing Hub]]**

> v6 부터 API 가 크게 바뀜. **v5 코드 → v6 마이그레이션 노트** 도 마지막에.

## 1. 설치

```bash
yarn add react-router-dom
# 또는
npm install react-router-dom
```

## 2. 최소 예제

```tsx
import { BrowserRouter, Routes, Route, Link } from 'react-router-dom';

const Home = () => <h1>Home</h1>;
const About = () => <h1>About</h1>;

function App() {
  return (
    <BrowserRouter>
      <nav>
        <Link to="/">Home</Link>
        <Link to="/about">About</Link>
      </nav>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
      </Routes>
    </BrowserRouter>
  );
}
```

→ `BrowserRouter` 가 최상위. `Routes` 안에 `Route` 들.

## 3. URL 파라미터 — `:id`

```tsx
<Route path="/users/:id" element={<UserDetail />} />
```

```tsx
import { useParams } from 'react-router-dom';

function UserDetail() {
  const { id } = useParams<{ id: string }>();
  // id 는 string (URL 은 항상 string).
  // 숫자가 필요하면 Number(id).

  return <h1>User #{id}</h1>;
}
```

## 4. Query string — `?key=value`

```tsx
// URL: /search?q=react&page=2
import { useSearchParams } from 'react-router-dom';

function Search() {
  const [params, setParams] = useSearchParams();

  const q = params.get('q') ?? '';
  const page = Number(params.get('page') ?? 1);

  const next = () => setParams({ q, page: String(page + 1) });

  return <button onClick={next}>다음</button>;
}
```

## 5. 프로그래매틱 이동 — `useNavigate`

```tsx
import { useNavigate } from 'react-router-dom';

function LoginButton() {
  const navigate = useNavigate();

  const handleLogin = async () => {
    await api.login();
    navigate('/dashboard');               // 일반 이동
    navigate('/dashboard', { replace: true });  // history replace (뒤로가기 안 됨)
    navigate(-1);                          // 뒤로 1 칸
  };

  return <button onClick={handleLogin}>로그인</button>;
}
```

→ submit 후 redirect, 모달 닫고 페이지 이동 등.

## 6. NavLink — 활성 상태 스타일링

```tsx
import { NavLink } from 'react-router-dom';

<NavLink
  to="/about"
  className={({ isActive }) => isActive ? 'active' : ''}
>
  About
</NavLink>
```

→ 현재 활성 route 면 isActive true. 메뉴의 selected 표시.

## 7. Nested Routes — 중첩

```tsx
<Routes>
  <Route path="/" element={<Layout />}>
    <Route index element={<Home />} />        // / (default child)
    <Route path="about" element={<About />} />
    <Route path="users" element={<Users />}>
      <Route path=":id" element={<UserDetail />} />
    </Route>
  </Route>
</Routes>
```

```tsx
// Layout.tsx — 부모는 <Outlet /> 으로 자식 위치 지정
import { Outlet } from 'react-router-dom';

function Layout() {
  return (
    <>
      <Header />
      <main><Outlet /></main>
      <Footer />
    </>
  );
}
```

→ 공통 레이아웃 + 페이지별 컨텐츠.

URL 매핑:
- `/` → Layout + Home
- `/about` → Layout + About
- `/users` → Layout + Users
- `/users/5` → Layout + Users + UserDetail

## 8. 404 처리

```tsx
<Route path="*" element={<NotFound />} />
```

→ 어느 path 도 안 맞으면 NotFound.

## 9. Private Route — 로그인 가드

```tsx
import { Navigate, useLocation } from 'react-router-dom';

function PrivateRoute({ children }: { children: ReactNode }) {
  const { user } = useAuth();
  const location = useLocation();

  if (!user) {
    return <Navigate to="/login" state={{ from: location }} replace />;
  }
  return <>{children}</>;
}

// 사용
<Routes>
  <Route path="/dashboard" element={
    <PrivateRoute><Dashboard /></PrivateRoute>
  } />
</Routes>
```

```tsx
// LoginPage — 로그인 성공 후 원래 가려던 곳으로
const location = useLocation();
const navigate = useNavigate();
const from = location.state?.from?.pathname ?? '/';
await api.login();
navigate(from, { replace: true });
```

→ [[../auth/auth]] 에서 자세히.

## 10. Lazy loading — 페이지별 chunk

```tsx
import { lazy, Suspense } from 'react';

const Dashboard = lazy(() => import('./pages/Dashboard'));
const Profile = lazy(() => import('./pages/Profile'));

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<Loading />}>
        <Routes>
          <Route path="/dashboard" element={<Dashboard />} />
          <Route path="/profile" element={<Profile />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}
```

→ 첫 화면 bundle 작게. [[../performance/code-splitting]] 자세히.

## 11. createBrowserRouter (v6.4+) — 새 권장 API

```tsx
import { createBrowserRouter, RouterProvider } from 'react-router-dom';

const router = createBrowserRouter([
  {
    path: '/',
    element: <Layout />,
    children: [
      { index: true, element: <Home /> },
      { path: 'about', element: <About /> },
      { path: 'users/:id', element: <UserDetail />, loader: userLoader },
    ],
  },
]);

function App() {
  return <RouterProvider router={router} />;
}
```

- **장점**: data API (loader / action), `errorElement` per route.
- **단점**: 학습 곡선 약간 더.
- `masterway-dev` 의 기존 코드는 `<BrowserRouter>` + `<Routes>` 위주.

## 12. loader / action — 데이터 fetching (v6.4+)

```tsx
async function userLoader({ params }) {
  const r = await fetch(`/api/users/${params.id}`);
  return r.json();
}

// route 정의
{ path: 'users/:id', element: <UserDetail />, loader: userLoader }

// UserDetail
import { useLoaderData } from 'react-router-dom';

function UserDetail() {
  const user = useLoaderData() as User;
  return <h1>{user.name}</h1>;
}
```

→ react-query 와의 관계: react-query 가 캐싱 / refetch / mutation 더 강력. 둘 다 사용 가능 (loader 에서 query.fetchQuery).

## 13. Scroll 복원

```tsx
import { useEffect } from 'react';
import { useLocation } from 'react-router-dom';

function ScrollToTop() {
  const { pathname } = useLocation();
  useEffect(() => {
    window.scrollTo(0, 0);
  }, [pathname]);
  return null;
}

// App.tsx
<BrowserRouter>
  <ScrollToTop />
  <Routes>...</Routes>
</BrowserRouter>
```

→ 페이지 이동 시 자동 스크롤 맨 위. (v6.4+ `ScrollRestoration` 컴포넌트 있음.)

## 14. answer-fe 의 흔한 패턴

```tsx
// App.tsx
<BrowserRouter>
  <Routes>
    <Route element={<MainLayout />}>
      <Route path="/" element={<HomePage />} />
      <Route path="/questions" element={<QuestionListPage />} />
      <Route path="/questions/:id" element={<QuestionDetailPage />} />

      <Route element={<PrivateRoute />}>
        <Route path="/mypage" element={<MyPage />} />
      </Route>
    </Route>

    <Route path="/login" element={<LoginPage />} />
    <Route path="*" element={<NotFound />} />
  </Routes>
</BrowserRouter>
```

```tsx
// PrivateRoute.tsx
function PrivateRoute() {
  const { user } = useAuth();
  return user ? <Outlet /> : <Navigate to="/login" replace />;
}
```

→ `<Outlet />` 패턴이 더 깔끔.

## 15. v5 → v6 주요 변경

| v5 | v6 |
| --- | --- |
| `<Switch>` | `<Routes>` |
| `<Route component={X}>` | `<Route element={<X />} />` |
| `<Redirect to=...>` | `<Navigate to=... />` |
| `useHistory().push` | `useNavigate()` |
| exact 명시 | 자동 exact |
| match.params | useParams |

## 16. 함정

1. **`Routes` 밖에 `Route`** — 모든 `Route` 는 `Routes` 안.
2. **`element={<X />}` 가 아니라 `element={X}`** — 컴포넌트 인스턴스 (JSX) 가 맞음.
3. **`:id` 가 항상 string** — 숫자 변환 잊지 말기.
4. **새로고침 시 404** — 서버의 SPA fallback 설정.
5. **`Navigate` 의 `replace` 누락** — login → dashboard 후 뒤로가기로 login 다시 보임.
6. **`useNavigate` 를 컴포넌트 밖에서 호출** — hook 규칙 위반.
7. **Nested route 에서 `Outlet` 누락** — 자식 안 보임.
8. **Search param 직접 mutate** — `params.set(...)` 만으로 안 됨. `setParams({...})` 호출 필요.

## 17. 외부 자료

- [react-router v6 공식](https://reactrouter.com/en/main)
- [v5 → v6 마이그레이션 가이드](https://reactrouter.com/en/main/upgrading/v5)

## 18. 관련

- [[routing]]
- [[../auth/auth]]
- [[../performance/code-splitting]]
