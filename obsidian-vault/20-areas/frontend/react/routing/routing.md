---
title: "Routing in React — SPA 의 페이지 전환"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, routing, react-router, hub, beginner]
---

# Routing in React

**[[../react|↑ React Hub]]**

> SPA 는 페이지 reload 없이 URL 만 바꿔 다른 컴포넌트를 렌더. = **client-side routing**.

## 1. 핵심 개념

| 옛 웹 (MPA) | SPA |
| --- | --- |
| URL 변경 = 서버 새 HTML 요청 | URL 변경 = JS 가 화면 갈아끼움 |
| 페이지 reload | reload 없음 |
| 모든 a 태그 | `<Link>` 컴포넌트 |

## 2. React 의 routing 선택지

- **react-router-dom** — de facto 표준. `masterway-dev/*-fe` 가 사용.
- **TanStack Router** — 새로운 type-safe router.
- **Next.js** / **Remix** — framework 내장 routing (file-based).
- **wouter** — 매우 작은 router (1KB).

→ 학습 우선: **react-router-dom v6**. [[react-router-v6]] 에서 자세히.

## 3. react-router-dom v6 의 빠른 데모

```tsx
import { BrowserRouter, Routes, Route, Link } from 'react-router-dom';

function App() {
  return (
    <BrowserRouter>
      <nav>
        <Link to="/">홈</Link>
        <Link to="/about">소개</Link>
      </nav>

      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
        <Route path="/users/:id" element={<UserDetail />} />
        <Route path="*" element={<NotFound />} />
      </Routes>
    </BrowserRouter>
  );
}
```

→ [[react-router-v6]] 에서 nested route, lazy, loader 등 전체.

## 4. Routing 의 핵심 작업들

- **URL 정의** — 어떤 path 가 어떤 컴포넌트.
- **이동 (Link / navigate)** — 페이지 이동.
- **파라미터 (params, query)** — URL 안 동적 값.
- **중첩 (nested)** — 레이아웃 + 하위 페이지.
- **가드** — 로그인 안 한 사용자 redirect.
- **404** — 매치 안 되는 path.
- **lazy** — 코드 분할로 페이지별 chunk.

## 5. 자기 프로젝트의 routing 구조

```bash
grep -rn "BrowserRouter\|createBrowserRouter\|<Route " ~/masterway-dev/answer-fe/src --include="*.tsx" | head -10
```

전형적 `answer-fe/src/App.tsx`:
```tsx
<BrowserRouter>
  <Routes>
    <Route path="/" element={<HomePage />} />
    <Route path="/questions" element={<QuestionList />} />
    <Route path="/questions/:id" element={<QuestionDetail />} />
    <Route path="/login" element={<LoginPage />} />
  </Routes>
</BrowserRouter>
```

## 6. SPA routing 의 주의 — 서버 설정

```
정적 호스팅 (S3 / Nginx) 에서 /about 직접 접속 시 404.
```

→ SPA 는 모든 path 를 `index.html` 로 fallback. 보통:
- Nginx: `try_files $uri /index.html;`
- Vercel / Netlify: 자동.
- S3 + CloudFront: 404 → 200 + index.html rewrite.

`vite preview` 는 자동 처리하지만 production 환경 설정 필수.

## 7. Hash vs Browser router

```tsx
// browser — /about (recommended)
<BrowserRouter>...</BrowserRouter>

// hash — /#/about (서버 설정 필요 없음, URL 더러움)
<HashRouter>...</HashRouter>
```

→ 대부분 BrowserRouter. 정적 호스팅 + fallback 설정 못 할 때만 Hash.

## 8. 학습 우선순위

1. **[[react-router-v6]]** — Routes / Route / Link / useNavigate / useParams.
2. **redirect / nested route**.
3. **lazy + Suspense 로 코드 분할**.
4. **route guard (private route)**.
5. **route data fetching** — v6.4+ loader, 또는 react-query 와 결합.

## 9. 다음 단계

- [[react-router-v6]] — 자세한 react-router 사용법

## 10. 관련

- [[../auth/auth|로그인과 route 보호]]
- [[../performance/code-splitting|페이지별 chunk 분리]]
- [[../server-state/react-query|페이지 별 데이터 fetching]]
