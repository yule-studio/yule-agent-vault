---
title: "Project Structure — 실 프로젝트 폴더 구조"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, project-structure, vite, beginner]
---

# Project Structure — 실 프로젝트 폴더 구조

**[[react|↑ React hub]]**

> `masterway-dev/answer-fe` 같은 실 프로젝트가 src 안을 어떻게 나누는가. 처음 큰 프로젝트 열면 막막한데, 이 패턴 익히면 어떤 React 코드 베이스든 1 시간 안에 파악 가능.

## 1. 최상위 폴더 — Vite 표준

```
my-app/
├── node_modules/         ← 라이브러리 (절대 안 만짐, git X)
├── public/               ← 정적 자산 (favicon, robots.txt)
├── src/                  ← ★ 99% 의 작업
├── .env.local            ← 환경 변수 (git X)
├── .gitignore
├── index.html            ← 진입점
├── package.json          ← 의존성 + 스크립트
├── tsconfig.json         ← TypeScript 설정
├── vite.config.ts        ← Vite 설정 (alias, proxy)
└── README.md
```

## 2. src/ 안의 표준 패턴 — `answer-fe` 구조

```
src/
├── main.tsx              ← React 시작점 (createRoot)
├── App.tsx               ← 최상위 컴포넌트
├── router.tsx            ← 라우팅 정의
├── queryClient.ts        ← react-query 설정
│
├── pages/                ← 라우트 별 페이지 컴포넌트
│   ├── HomePage.tsx
│   ├── LoginPage.tsx
│   ├── DashboardPage.tsx
│   └── ...
│
├── components/           ← 재사용 컴포넌트
│   ├── atom/             ← 가장 작은 단위 (Button, Input)
│   ├── molecules/        ← 작은 조합 (FormField, Card)
│   ├── layout/           ← 레이아웃 (Header, Sidebar, Footer)
│   ├── lounge/           ← 도메인별 묶음
│   ├── editor/           ← 에디터 관련 (CKEditor wrapper)
│   ├── pagination/       ← 페이지네이션
│   └── context-menu/     ← 우클릭 메뉴
│
├── hooks/                ← 커스텀 hook
│   ├── useAuth.ts
│   ├── useDebounce.ts
│   └── ...
│
├── store/                ← 글로벌 상태 (jotai / recoil)
│   ├── authStore.ts
│   ├── userStore.ts
│   └── ...
│
├── auth/                 ← 인증 로직 / guard
│   └── ProtectedRoute.tsx
│
├── util/                 ← 순수 함수 utility
│   ├── api/              ← axios 인스턴스, API client
│   ├── pdf/              ← PDF 생성 유틸
│   ├── vaildation/       ← (오타 그대로 — 실 프로젝트 자주 있음)
│   └── format.ts
│
├── types/                ← TypeScript 타입 정의
│   ├── assessment.ts
│   ├── user.ts
│   └── ...
│
├── styles/               ← 글로벌 CSS / theme
│   ├── globalStyle.ts
│   └── reset.css
│
├── theme/                ← MUI / styled-components 의 theme
│   └── theme.ts
│
├── meta/                 ← SEO / head meta
│
├── assets/               ← 이미지 / 아이콘 / 폰트
│   ├── images/
│   ├── icon/
│   └── picture/
│
└── fonts/                ← 폰트 파일
```

## 3. Atomic Design 폴더 패턴

`components/atom/`, `molecules/` 같은 이름은 **Atomic Design** 방법론.

| 단위 | 의미 | 예 |
| --- | --- | --- |
| **Atom** | 가장 작은 UI | Button, Input, Label, Icon |
| **Molecule** | Atom 의 작은 조합 | FormField (Label + Input), SearchBar |
| **Organism** | Molecule + Atom 조합 | Header, ProductCard, CommentList |
| **Template** | 페이지 레이아웃 (데이터 X) | HomePageLayout, DashboardLayout |
| **Page** | 실 데이터 채운 페이지 | HomePage, ProductPage |

→ 실 프로젝트는 100% 따르지 않고 도메인별 폴더 (`lounge/`, `editor/`) 와 섞임.

## 4. 도메인 기반 vs 타입 기반

### 타입 기반 (`answer-fe` 같은 큰 프로젝트의 답)
```
src/
├── components/        ← 모든 컴포넌트 한 곳
├── hooks/             ← 모든 hook 한 곳
├── pages/             ← 모든 페이지 한 곳
└── types/             ← 모든 타입 한 곳
```
- 처음에 단순. 작은 프로젝트에 좋음.
- 도메인 커지면 한 폴더에 100+ 파일 → 찾기 어려움.

### 도메인 기반 (큰 프로젝트의 답)
```
src/
├── features/
│   ├── auth/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── api.ts
│   │   ├── types.ts
│   │   └── index.ts
│   ├── product/
│   │   ├── components/
│   │   ├── hooks/
│   │   └── ...
│   └── order/
└── shared/             ← 도메인 무관 공통
    ├── components/
    ├── hooks/
    └── utils/
```
- 도메인 별 응집도 ↑.
- "이 기능 어디 있지?" 가 명확.
- 100+ 파일 프로젝트의 표준.

### 어느 쪽?

- **소규모 / 학습 프로젝트** = 타입 기반 (단순).
- **5+ 도메인 / 10명+ 팀** = 도메인 기반 (Feature Sliced Design, "FSD" 라고도).

`answer-fe` 는 처음 타입 기반으로 시작했다가 자연스럽게 components 안 도메인별 폴더 (`lounge/`, `editor/`, `seteuk/`) 가 자라난 형태.

## 5. 파일명 규칙

### React 컴포넌트
- **PascalCase**: `Button.tsx`, `UserProfile.tsx`, `LoginPage.tsx`.
- 한 파일 한 컴포넌트 (대원칙).

### Hook
- **camelCase + `use` 접두**: `useAuth.ts`, `useDebounce.ts`, `useLocalStorage.ts`.
- 반드시 `use` 로 시작해야 React 가 hook 으로 인식.

### Utility 함수
- **camelCase**: `formatDate.ts`, `parseQuery.ts`.

### 타입 / 인터페이스
- **camelCase 파일**, **PascalCase 타입**:
  ```ts
  // user.ts
  export interface User { id: number; name: string }
  export type UserRole = 'admin' | 'user';
  ```

### CSS / Style
- **컴포넌트 같은 이름 + 확장자**: `Button.module.css`, `Button.styled.ts`.
- styled-components 의 경우 컴포넌트 옆에 두기도.

## 6. 한 컴포넌트 폴더 패턴

복잡한 컴포넌트는 폴더로:

```
components/UserProfile/
├── index.tsx              ← 메인 컴포넌트
├── UserProfile.tsx        ← (또는 위 대신)
├── UserProfile.styled.ts  ← styled-components
├── UserProfile.test.tsx   ← 테스트
├── UserProfile.stories.tsx ← Storybook (있으면)
├── types.ts               ← 컴포넌트 내부 타입
├── hooks.ts               ← 컴포넌트 내부 hook
└── components/            ← 자식 컴포넌트
    ├── Avatar.tsx
    └── Bio.tsx
```

장점: 컴포넌트 + 스타일 + 테스트 + 자식 한 곳.
import: `import UserProfile from 'components/UserProfile'` (index 가 entry).

## 7. import 절대 경로 (alias)

```tsx
// ❌ 상대 경로 헬
import Button from '../../../components/atom/Button';

// ✅ 절대 경로
import Button from '@/components/atom/Button';
```

`vite.config.ts` 에 `@` = `src/` 설정:
```ts
import path from 'path';

export default {
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
};
```

`tsconfig.json` 에도 같이:
```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

자세한 건 [[configuration]].

## 8. 실 프로젝트 분석 — `answer-fe` src 핵심 5 파일

```bash
cd ~/masterway-dev/answer-fe
ls src/
```

| 파일 | 역할 |
| --- | --- |
| `main.tsx` | ReactDOM 시작점 |
| `App.tsx` | 최상위 + Provider 들 wrap |
| `router.tsx` | 모든 라우트 정의 |
| `queryClient.ts` | react-query QueryClient 인스턴스 |
| `atoms.js` | jotai atom 글로벌 정의 |

### `App.tsx` 의 전형적 구조

```tsx
function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <RecoilRoot>                   // 또는 <Provider> (jotai)
        <ThemeProvider theme={theme}>
          <GlobalStyle />
          <BrowserRouter>
            <Router />               // src/router.tsx
          </BrowserRouter>
        </ThemeProvider>
      </RecoilRoot>
    </QueryClientProvider>
  );
}
```

→ Provider 들이 양파 껍질처럼 wrap. 안쪽 컴포넌트가 각 Provider 의 context 사용.

## 9. 실 프로젝트 따라하기

```bash
# answer-fe 의 src 구조 시각화
cd ~/masterway-dev/answer-fe/src
tree -L 2 -d                 # 디렉토리만, 2 단계
```

→ 위 §2 와 같은 구조 보일 것.

```bash
# 컴포넌트 개수
find src/components -name "*.tsx" | wc -l

# 페이지 개수
find src/pages -name "*.tsx" | wc -l
```

## 10. 함정

1. **`src/components` 한 곳에 100+ 파일** — 도메인 별 sub-folder 분리.
2. **`pages/` 와 `components/` 의 경계 모호** — page = 라우트 매핑, component = 재사용 가능 단위.
3. **`utils/` 가 잡탕** — 도메인 logic 까지 utils 에 들어감. utils 는 **순수 함수만**.
4. **상대 경로 `../../../`** — alias 설정 (`@`) 필수.
5. **`index.ts` re-export 폭주** — barrel file 이 너무 많으면 tree-shaking 깨짐.
6. **컴포넌트 이름이 파일명과 다름** — `Button.tsx` 안 `export default function Btn()` 같은. 디버깅 헬.
7. **`assets/` 와 `public/` 혼동** — public = 빌드 후 그대로, assets = import 후 번들 (해시 + 압축).

## 11. 다음 단계

- [[configuration]] — vite / tsconfig / eslint
- [[core-concepts/components-and-jsx]] — 컴포넌트 깊이

## 12. 관련

- [[react]]
- [[getting-started]]
- [[core-concepts/core-concepts]]
