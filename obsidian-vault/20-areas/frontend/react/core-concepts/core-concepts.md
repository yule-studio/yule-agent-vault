---
title: "Core Concepts — React 핵심 개념 hub"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, core-concepts, hub]
---

# Core Concepts — React 핵심 개념 hub

**[[../react|↑ React hub]]**

> React 의 모든 것은 이 6 개 노트로 압축됨. 이 파일들만 익혀도 자기 프로젝트의 90% 코드 읽을 수 있음.

## 1. 학습 순서

1. **[[components-and-jsx]]** — 컴포넌트는 함수. JSX 는 화면 그리기.
2. **[[props-and-state]]** — 부모 → 자식 데이터 전달 + 변하는 상태.
3. **[[hooks-essentials]]** — useState / useEffect / useRef / useMemo / useCallback.
4. **[[conditional-and-list-rendering]]** — 조건부 화면 + 배열 → 화면 N 개.
5. **[[events-and-synthetic]]** — onClick / onChange + 이벤트의 React 특이점.
6. **[[forms-and-controlled-inputs]]** — 입력 폼.

## 2. React 의 큰 그림 — 한 줄 요약

```
컴포넌트(=함수) → JSX(=화면) → 상태(=데이터) 변경 시 → 자동 re-render → 화면 갱신
```

이 한 줄을 이해하면 React 의 80%.

## 3. 한 컴포넌트의 생명 흐름

```
1. 컴포넌트 함수 호출 (mount)
   ↓
2. JSX return
   ↓
3. React 가 DOM 그림
   ↓
4. useEffect 실행 (mount 직후)
   ↓
5. 사용자 이벤트 (클릭 등) → setState
   ↓
6. 컴포넌트 함수 다시 호출 (re-render)
   ↓
7. JSX 다시 return
   ↓
8. React 가 diff 비교 → 바뀐 부분만 DOM 갱신
   ↓
9. useEffect 의 dependencies 가 바뀌었으면 다시 실행
   ↓
(반복)
```

→ "다시 호출되는 함수" 라는 사상이 핵심. 함수가 매 render 마다 처음부터 다시 실행됨.

## 4. 자주 헷갈리는 5 가지

### 1. "왜 변수 바꾸면 화면 안 바뀌어?"
```tsx
let count = 0;
function App() {
  return <button onClick={() => count++}>{count}</button>;  // ❌
}
```
- React 는 **상태가 변했다고 알려야** re-render.
- 그냥 변수 변경은 알 수 없음.
- **`useState`** 사용 필요. → [[props-and-state]]

### 2. "useEffect 가 두 번 실행돼!"
- 개발 모드 `<React.StrictMode>` 가 일부러 두 번 실행 → 버그 잡기 용.
- production 에서는 1 번만.
- → [[hooks-essentials]] `useEffect` 부분.

### 3. "props 를 자식에서 못 바꿔!"
- props 는 **읽기 전용**.
- 자식이 부모 데이터 바꾸려면 부모가 callback prop 으로 내려줘야.
- → [[props-and-state]]

### 4. "map 의 key warning 이 뭐야?"
```tsx
{items.map((item) => <li>{item.name}</li>)}    // ⚠️ key 누락
{items.map((item) => <li key={item.id}>{item.name}</li>)}  // ✅
```
- React 가 list 의 어떤 item 이 추가/변경/삭제 됐는지 추적용.
- → [[conditional-and-list-rendering]]

### 5. "form 입력값이 안 바뀌어!"
- "controlled component" 패턴 미적용.
- → [[forms-and-controlled-inputs]]

## 5. JSX 의 mental model

JSX 는 **JS 표현식이 가능한 HTML 처럼 생긴 것**:

```tsx
function Greeting({ name, isLoggedIn }) {
  return (
    <div>
      {isLoggedIn ? (             // 조건부
        <h1>안녕, {name}!</h1>     // 변수 삽입
      ) : (
        <h1>로그인 해주세요</h1>
      )}
    </div>
  );
}
```

- `{ }` 안은 JS 식.
- 안에 함수 호출, 변수, 삼항연산자, map 모두 가능.

## 6. 컴포넌트의 두 종류

### Function Component (현대 표준)
```tsx
function Button({ onClick, children }) {
  return <button onClick={onClick}>{children}</button>;
}
```

### Class Component (옛 방식, 거의 안 씀)
```tsx
class Button extends React.Component {
  render() { return <button onClick={this.props.onClick}>{this.props.children}</button>; }
}
```

→ **2026 의 React 는 100% function component**. class 는 옛 코드에만.

## 7. 자기 프로젝트와 비교

`masterway-dev/answer-fe/src/components/atom` 같은 폴더 가서 가장 단순한 컴포넌트 5 개 읽기:

```bash
cd ~/masterway-dev/answer-fe/src/components/atom
ls *.tsx | head -5
cat Button.tsx     # 또는 유사 단순 컴포넌트
```

처음에는 모르는 코드가 90% 일 것. 이 hub 의 6 노트 끝나면 그 90% 가 30% 로 줄어요.

## 8. 관련

- [[../react|↑ React hub]]
- [[../typescript/typescript|TypeScript]] — JSX 와 결합되어 자주 등장
- [[../routing/routing|Routing]] — 컴포넌트의 라우팅
