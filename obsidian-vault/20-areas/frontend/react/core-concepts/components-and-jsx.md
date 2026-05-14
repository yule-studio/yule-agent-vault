---
title: "Components and JSX — 컴포넌트는 함수"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, components, jsx, beginner]
---

# Components and JSX — 컴포넌트는 함수

**[[core-concepts|↑ Core Concepts]]**

> React 의 가장 기본 단위. 컴포넌트 = 함수, JSX = 그 함수가 그릴 화면.

## 1. 컴포넌트 = JSX 를 return 하는 함수

```tsx
function Welcome() {
  return <h1>안녕!</h1>;
}
```

### 사용법
```tsx
function App() {
  return (
    <div>
      <Welcome />     {/* 컴포넌트는 <PascalCase /> */}
      <Welcome />
      <Welcome />
    </div>
  );
}
```

→ 같은 컴포넌트를 N 번 쓸 수 있음. **재사용 단위**.

### 명명 규칙 (필수)
- **PascalCase**: `Welcome`, `UserProfile`, `LoginPage`.
- 소문자로 시작하면 React 가 HTML 태그로 오해.
- 함수 이름 = 컴포넌트 이름.

## 2. JSX — JavaScript 안의 HTML 같은 문법

### JSX 의 정체

```tsx
const greeting = <h1 className="title">안녕</h1>;
```

위 코드는 사실 JS 가 아님. Vite/Babel 이 컴파일:

```js
const greeting = React.createElement('h1', { className: 'title' }, '안녕');
```

→ **결국 JS 객체**. JSX 는 가독성을 위한 syntactic sugar.

### JSX 가 JS 와 다른 5 가지

| | HTML / 일반 | JSX |
| --- | --- | --- |
| 1 | `class="..."` | `className="..."` |
| 2 | `for="..."` | `htmlFor="..."` |
| 3 | `onclick` | `onClick` (camelCase) |
| 4 | `<br>` | `<br />` (반드시 self-close) |
| 5 | `<input>` | `<input type="text" />` |

### `{ }` — JS 표현식 삽입

```tsx
function Hello({ name }) {
  const today = new Date().toLocaleDateString();
  return (
    <div>
      <h1>안녕, {name}!</h1>
      <p>오늘은 {today}</p>
      <p>점수: {Math.floor(Math.random() * 100)}</p>
      <p>이름 길이: {name.length}</p>
    </div>
  );
}
```

`{ }` 안에는 **JS 표현식** (값을 만드는 식) 만. 문장 (statement) 은 X:
- ✅ `{a + b}`, `{name}`, `{user.name}`, `{a > b ? 'yes' : 'no'}`, `{items.map(...)}`
- ❌ `{if (a) ...}`, `{for (...) ...}` — 삼항연산자 / map 으로 치환.

## 3. 한 컴포넌트 = 한 element return

```tsx
// ❌ 두 element 동시 return
function Bad() {
  return <h1>제목</h1><p>본문</p>;
}

// ✅ 감싸기
function Good1() {
  return <div><h1>제목</h1><p>본문</p></div>;
}

// ✅ Fragment (불필요 div 안 만들고 싶을 때)
function Good2() {
  return <><h1>제목</h1><p>본문</p></>;
}

// ✅ React.Fragment
function Good3() {
  return (
    <React.Fragment key={...}>
      <h1>제목</h1>
      <p>본문</p>
    </React.Fragment>
  );
}
```

→ Fragment (`<>...</>`) 가 가장 흔함. 불필요한 div 안 만듬.

## 4. children — 컴포넌트 안에 또 컴포넌트

```tsx
function Card({ children }) {
  return <div className="card">{children}</div>;
}

function App() {
  return (
    <Card>
      <h1>제목</h1>           {/* 이게 children */}
      <p>본문</p>             {/* 이것도 */}
    </Card>
  );
}
```

→ `children` 은 React 의 특별 prop. 컴포넌트 태그 사이 모든 것.

### children 활용
- **Layout 컴포넌트**: Header / Sidebar 안 컨텐츠.
- **Modal / Drawer**: 안에 넣는 컨텐츠.
- **List / Card / Section**: 어떤 컨텐츠도 wrap.

## 5. 컴포넌트 분리 — 언제 / 어떻게

### 언제 분리?
- **반복** — 같은 UI 가 2 번 이상 나타날 때.
- **복잡** — 한 함수가 100 줄 넘으면.
- **재사용** — 다른 페이지에서도 쓸 가능성.
- **테스트** — 작은 단위 테스트 원할 때.

### 어떻게?

```tsx
// Before — 거대한 페이지
function HomePage() {
  return (
    <div>
      <header>
        <img src="..." />
        <nav>...</nav>
      </header>
      <main>
        <section>...</section>
        <section>...</section>
      </main>
      <footer>...</footer>
    </div>
  );
}
```

```tsx
// After — 분리
function HomePage() {
  return (
    <div>
      <Header />
      <main>
        <HeroSection />
        <FeatureSection />
      </main>
      <Footer />
    </div>
  );
}

function Header() { return <header>...</header>; }
function HeroSection() { return <section>...</section>; }
// ...
```

## 6. 한 파일 한 컴포넌트 vs 한 파일 N 컴포넌트

```tsx
// 한 파일 = 1 메인 컴포넌트 + 자식 (선호)
// UserCard.tsx
function Avatar({ src }) { return <img src={src} />; }   // 내부 helper
function Bio({ text }) { return <p>{text}</p>; }          // 내부 helper

export default function UserCard({ user }) {
  return (
    <div>
      <Avatar src={user.avatar} />
      <Bio text={user.bio} />
    </div>
  );
}
```

→ 같은 파일 내 작은 helper 컴포넌트 OK.

## 7. PropTypes 는 잊어도 됨

```tsx
// ❌ 옛 방식 (PropTypes)
import PropTypes from 'prop-types';
Component.propTypes = { name: PropTypes.string };

// ✅ 현대 방식 (TypeScript)
function Component({ name }: { name: string }) {
  return <h1>{name}</h1>;
}
```

→ TypeScript 가 PropTypes 의 역할.

## 8. 함수 컴포넌트 vs 화살표 함수 컴포넌트

```tsx
// 1. 함수 선언식
function Button(props) {
  return <button>{props.label}</button>;
}

// 2. 화살표 함수 (또는 default export)
const Button = (props) => {
  return <button>{props.label}</button>;
};

// 3. 가장 짧게 (return 한 줄)
const Button = (props) => <button>{props.label}</button>;
```

→ 어느 쪽이든 동작 같음. **팀 컨벤션 따라 통일**.

`masterway-dev/answer-fe` 는 보통 `const X = () => { ... }` 형태.

## 9. Default vs Named export

```tsx
// default export — 1 컴포넌트 1 파일 시 흔함
export default function Button() { ... }

// import 시 이름 자유
import Btn from './Button';
import MyButton from './Button';

// named export — 한 파일 N 개
export function Button() { ... }
export function IconButton() { ... }

// import 시 정확한 이름
import { Button, IconButton } from './buttons';
```

→ default 가 한국 / 일본에서 흔함. named 가 영문 community 권장. **팀 컨벤션**.

## 10. Self-closing 규칙

```tsx
{/* 모든 비어 있는 태그는 self-close */}
<img src="..." />
<input type="text" />
<br />
<hr />
<MyComponent />

{/* children 있으면 일반 형식 */}
<MyComponent>some children</MyComponent>
```

## 11. 자기 프로젝트 보기 — 가장 단순한 컴포넌트

```bash
cd ~/masterway-dev/answer-fe/src/components/atom
ls -1 *.tsx | head -3
```

랜덤 1 개 골라서:
```bash
cat Button.tsx     # 또는 보이는 파일
```

→ 위 §1-10 의 패턴 모두 보일 것. 처음에는 모르는 부분 (styled-components, props type 등) 이 많지만 컴포넌트 = 함수, return 안 JSX 라는 큰 틀만 잡으면 OK.

## 12. 함정

1. **소문자 컴포넌트 이름** — `<button />` 은 HTML, `<Button />` 이 컴포넌트.
2. **`return` 빠뜨림** — 화살표 함수의 `{ }` 안 return 안 쓰면 undefined return.
   ```tsx
   const X = () => { <h1>...</h1> }       // ❌ undefined return
   const X = () => { return <h1>...</h1> } // ✅
   const X = () => <h1>...</h1>           // ✅ 즉시 return
   ```
3. **`<>` Fragment 의 key 못 쓰는 케이스** — list 의 fragment 면 `<React.Fragment key={i}>` 사용.
4. **HTML 의 `class` / `for`** 그대로 쓰기 — `className` / `htmlFor`.
5. **JSX 안 boolean** — `{false}` / `{null}` 은 안 보이지만 `{0}` 은 "0" 보임. `{count !== 0 && <div>...</div>}` 패턴.
6. **컴포넌트 내부 함수 정의** — 매 render 마다 새 함수 생성. 자주 호출되는 자식 prop 으로 내려가면 자식이 re-render. `useCallback` 으로 최적화 (필요 시).

## 13. 다음 단계

- [[props-and-state]] — 데이터 전달과 상태
- [[hooks-essentials]] — useState / useEffect

## 14. 관련

- [[core-concepts]]
- [[../typescript/component-types|컴포넌트 타입]]
