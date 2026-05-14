---
title: "Getting Started — React 처음 시작"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, getting-started, vite, beginner]
---

# Getting Started — React 처음 시작

**[[react|↑ React hub]]**

> 아무 것도 모르는 상태에서 React 프로젝트 1 개를 띄우기까지. 한 번도 React 안 써봤다면 이 파일부터.

## 1. React 가 뭐예요?

**React** = Facebook (Meta) 이 만든 JavaScript 라이브러리. "**컴포넌트**" 라는 작은 조각으로 화면을 만든다.

```
[버튼] = 컴포넌트
[버튼 + 텍스트 + 입력창] = 더 큰 컴포넌트
[헤더 + 본문 + 푸터] = 페이지
[페이지 + 페이지 + 페이지] = 앱
```

→ 작은 조각 (컴포넌트) 을 조립해서 큰 화면을 만든다는 사상.

### React 가 아닌 옛 방식 (jQuery 시대)

```html
<button id="btn">눌러</button>
<script>
  document.getElementById("btn").addEventListener("click", () => {
    document.getElementById("count").innerText = "1";
  });
</script>
```

- DOM 을 직접 만짐. 화면 / 데이터 / 이벤트가 다 흩어짐.
- 페이지 커지면 관리 불가.

### React 방식

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

- 화면 + 데이터 + 이벤트가 한 컴포넌트 안에.
- `count` 가 바뀌면 React 가 자동으로 화면 다시 그림.

## 2. 사전 준비

### 설치할 것

| 도구 | 의미 | 설치 |
| --- | --- | --- |
| **Node.js** (LTS) | JavaScript 실행 환경 + npm 패키지 매니저 | https://nodejs.org 또는 `nvm` |
| **Yarn** (선택) | 더 빠른 패키지 매니저 | `npm install -g yarn` |
| **VS Code** | 코드 편집기 | https://code.visualstudio.com |
| **Git** | 버전 관리 | https://git-scm.com |
| **Chrome** | 브라우저 + DevTools | — |

### VS Code 확장

- **ES7+ React/Redux/React-Native snippets** (rfc, useState 자동완성)
- **Prettier — Code formatter** (코드 자동 정렬)
- **ESLint** (코드 문법 검사)
- **Auto Rename Tag** (`<div>` 한 쪽 바꾸면 닫는 태그도 자동)
- **Path Intellisense** (파일 경로 자동완성)

### Node.js 버전 확인

```bash
node --version    # v20.x.x 이상 권장
npm --version     # 10.x.x 이상
```

`v18` 미만이면 업그레이드. nvm 으로:

```bash
nvm install 20
nvm use 20
```

## 3. 첫 React 프로젝트 만들기 — Vite

**Vite (비트)** = 가장 빠른 React 개발 환경. `masterway-dev/*-fe` 모든 프로젝트가 Vite 사용.

```bash
# 프로젝트 만들기
npm create vite@latest my-first-app -- --template react-ts

# 들어가서 설치
cd my-first-app
npm install      # 또는 yarn install

# 개발 서버 시작
npm run dev      # 또는 yarn dev
```

→ 브라우저 자동으로 http://localhost:5173 열림. **Vite + React 18 + TypeScript** 가 완성된 상태.

### 결과 폴더

```
my-first-app/
├── node_modules/           ← 라이브러리들 (절대 git 에 안 올림)
├── public/                 ← 정적 자산 (favicon 등)
├── src/                    ← ★ 우리가 작업할 곳
│   ├── App.tsx             ← 메인 컴포넌트
│   ├── main.tsx            ← 시작점
│   └── index.css           ← 글로벌 스타일
├── index.html              ← HTML 진입점
├── package.json            ← 의존성 / 스크립트
├── tsconfig.json           ← TypeScript 설정
└── vite.config.ts          ← Vite 설정
```

### 첫 코드 분석 — `src/main.tsx`

```tsx
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App'
import './index.css'

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
)
```

- `index.html` 안 `<div id="root">` 에 `<App />` 을 그림.
- `React.StrictMode` = 개발 중 잠재 버그 잡아주는 wrapper (production 영향 없음).

### 첫 컴포넌트 — `src/App.tsx`

```tsx
import { useState } from 'react'

function App() {
  const [count, setCount] = useState(0)

  return (
    <div>
      <h1>안녕, React!</h1>
      <button onClick={() => setCount(count + 1)}>
        count = {count}
      </button>
    </div>
  )
}

export default App
```

이게 React 의 모든 것의 90%:
- **function** = 컴포넌트.
- **return** 안 JSX = 화면.
- **useState** = 변하는 값 (상태).
- **onClick** = 이벤트 핸들러.

## 4. 라이브 서버

```bash
npm run dev
```

→ 코드 저장하면 **자동으로 브라우저가 새로고침** (Hot Module Replacement, HMR).

**처음 보면 신기한 점**:
- `count` 버튼 누르면 숫자 올라감.
- 코드의 "안녕, React!" 를 "Hello!" 로 바꾸면 즉시 화면이 바뀜.
- 새로고침 안 해도 됨.

## 5. JSX 가 뭐예요?

```tsx
return <h1>안녕</h1>;
```

위 코드는 사실 JavaScript 가 아님. **JSX** = JavaScript + XML.

컴파일 후:
```js
return React.createElement('h1', null, '안녕');
```

- 브라우저는 JSX 못 읽음. Vite / Babel 이 자동 변환.
- "JS 안에 HTML 같은 문법" 을 쓸 수 있게.

### JSX 의 규칙 (처음 만나는 5 가지)

1. **return 안 하나의 부모 element**:
   ```tsx
   // ❌ 두 element 동시 return 못 함
   return <h1>a</h1><h2>b</h2>

   // ✅ 감싸기
   return <div><h1>a</h1><h2>b</h2></div>

   // ✅ 또는 Fragment
   return <><h1>a</h1><h2>b</h2></>
   ```

2. **JS 표현식은 `{ }` 안에**:
   ```tsx
   const name = "Alice";
   return <p>안녕, {name}님!</p>
   ```

3. **`class` 대신 `className`** (class 는 JS 예약어):
   ```tsx
   return <div className="container">...</div>
   ```

4. **속성은 camelCase**:
   - `onclick` → `onClick`
   - `onchange` → `onChange`
   - `tabindex` → `tabIndex`

5. **self-close 필수**:
   ```tsx
   <img src="..." />        // ✅
   <input type="text" />    // ✅
   <br />                   // ✅
   ```

## 6. 자기 프로젝트와 비교 — `masterway-dev/uready-fe`

`uready-fe` 는 가장 단순한 React 19 프로젝트입니다. 직접 비교:

```bash
cd ~/masterway-dev/uready-fe
cat package.json | head -30
ls src/
```

- 같은 Vite + React 구조.
- 차이: React 18 vs 19. 거의 동일하게 동작.

## 7. npm vs yarn 차이

| | npm | yarn |
| --- | --- | --- |
| 설치 | Node 와 함께 자동 | 별도 설치 |
| 속도 | 보통 | 약간 빠름 |
| Lock 파일 | `package-lock.json` | `yarn.lock` |
| 명령어 | `npm install`, `npm run dev` | `yarn`, `yarn dev` |

→ **한 프로젝트에서는 하나만 쓰기**. `masterway-dev` 의 프로젝트는 보통 yarn. lock 파일 종류로 판단.

```bash
# 어떤 매니저인지 확인
ls package-lock.json yarn.lock pnpm-lock.yaml 2>/dev/null
```

## 8. 자주 마주치는 첫 에러

### "Module not found"
- `npm install` 안 했음. 다시 실행.

### "Cannot find module 'react'"
- `package.json` 의 `dependencies` 에 react 없음 또는 `node_modules` 손상.
- 해결: `rm -rf node_modules package-lock.json && npm install`

### "Unexpected token <"
- `.js` 안에 JSX 작성. `.jsx` 또는 `.tsx` 로 확장자 변경.

### "ENOENT no such file"
- 경로 오타. `./` vs `../` 확인.

### Port 5173 already in use
- 다른 Vite 가 같은 port 사용 중. `lsof -i :5173` 로 찾아서 죽이거나 `npm run dev -- --port 5174`.

## 9. 다음 단계

- [[project-structure]] — `answer-fe` 같은 실 프로젝트의 폴더 구조 이해
- [[configuration]] — vite / tsconfig / eslint / prettier
- [[core-concepts/components-and-jsx]] — 컴포넌트 / JSX 깊이

## 10. 관련

- [[react]]
- [[core-concepts/core-concepts]]
