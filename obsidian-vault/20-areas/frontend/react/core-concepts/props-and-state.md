---
title: "Props and State — 데이터의 두 흐름"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, props, state, beginner]
---

# Props and State — 데이터의 두 흐름

**[[core-concepts|↑ Core Concepts]]**

> Props = 부모가 내려주는 값 (read-only). State = 컴포넌트 자기 자신이 가진 값 (변경 가능).

## 1. Props — 부모 → 자식

```tsx
function Greeting({ name, age }) {
  return <p>{name} 님은 {age} 살.</p>;
}

function App() {
  return <Greeting name="유철" age={30} />;
}
```

- 부모 (`App`) 가 `name`, `age` 를 자식 (`Greeting`) 에 **prop 으로 전달**.
- 자식은 prop 을 **읽기만**. 수정 금지.

### Props 의 모든 타입

```tsx
function MyComp({
  name,                   // string
  age,                    // number
  isActive,               // boolean
  items,                  // array
  user,                   // object
  onClick,                // function
  children,               // ReactNode
  icon,                   // JSX element
}) {
  // ...
}

<MyComp
  name="유철"
  age={30}                {/* number 는 { } */}
  isActive={true}
  items={['a', 'b']}
  user={{ id: 1, name: '유철' }}    {/* 객체 = { { ... } } 이중 중괄호 */}
  onClick={() => console.log('hi')}
  icon={<Icon />}
>
  자식 내용                {/* children */}
</MyComp>
```

### Destructuring (구조 분해)

```tsx
// 1. props 객체로 받기
function Card(props) {
  return <h1>{props.title}</h1>;
}

// 2. destructure (선호)
function Card({ title }) {
  return <h1>{title}</h1>;
}

// 3. 기본값
function Card({ title = '제목 없음' }) {
  return <h1>{title}</h1>;
}

// 4. rest
function Card({ title, ...rest }) {
  return <div {...rest}><h1>{title}</h1></div>;
}
```

## 2. State — 컴포넌트 자신의 변수

```tsx
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);
  //     ↑ 현재 값  ↑ 값 바꾸는 함수  ↑ 초기값

  return (
    <div>
      <p>{count}</p>
      <button onClick={() => setCount(count + 1)}>+1</button>
    </div>
  );
}
```

- `count` 가 바뀌면 React 가 **자동으로 컴포넌트 re-render**.
- `setCount(...)` 호출하지 않고 `count = ...` 직접 할당 ❌ → 변경 안 감지.

### useState 의 모든 타입

```tsx
const [name, setName] = useState('');           // string
const [age, setAge] = useState(0);              // number
const [isOpen, setIsOpen] = useState(false);    // boolean
const [items, setItems] = useState([]);          // array
const [user, setUser] = useState(null);          // null 또는 객체
const [data, setData] = useState<User | null>(null);  // TS 명시
```

## 3. State 의 핵심 규칙 — Immutable

```tsx
const [items, setItems] = useState([1, 2, 3]);

// ❌ 직접 mutate — React 가 변경 감지 못함
items.push(4);
setItems(items);   // 같은 배열 참조 → no re-render

// ✅ 새 배열 만들기
setItems([...items, 4]);                // 추가
setItems(items.filter(x => x !== 2));   // 삭제
setItems(items.map(x => x === 2 ? 20 : x)); // 수정
```

```tsx
const [user, setUser] = useState({ name: '유철', age: 30 });

// ❌
user.age = 31;
setUser(user);

// ✅ 새 객체
setUser({ ...user, age: 31 });
```

→ **React 의 변경 감지 = 참조 비교 (===)**. 같은 객체면 변경 없음으로 판단.

## 4. State 의 비동기성

```tsx
function Counter() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setCount(count + 1);
    console.log(count);   // 아직 0! (setCount 는 비동기)
  };
  // ...
}
```

→ `setCount` 직후 `count` 는 **아직 옛 값**. React 는 batch + 비동기로 update.

### 함수형 update — 이전 값 기반

```tsx
const handleClick = () => {
  setCount(count + 1);    // 이전 값 의존
  setCount(count + 1);    // 같은 값 또 +1 → 실제로는 1 만 증가
};

// ✅ 함수형
const handleClick = () => {
  setCount(prev => prev + 1);   // 0 → 1
  setCount(prev => prev + 1);   // 1 → 2 (정확히 2 증가)
};
```

→ 이전 state 기반으로 next state 만들 때 항상 **함수형 update**.

## 5. Props vs State — 비교표

| | Props | State |
| --- | --- | --- |
| 누구 정의 | 부모 컴포넌트 | 자신 |
| 수정 가능 | ❌ Read-only | ✅ setX 로만 |
| 변경 시 re-render | 자식 re-render | 자신 + 자식 re-render |
| 비유 | 함수의 인자 | 함수 안 local var |

## 6. Props "drilling" 문제

```tsx
function App() {
  const [user, setUser] = useState({ name: '유철' });
  return <Page user={user} />;
}

function Page({ user }) {
  return <Sidebar user={user} />;
}

function Sidebar({ user }) {
  return <Profile user={user} />;
}

function Profile({ user }) {
  return <p>{user.name}</p>;
}
```

→ `user` 가 5 단계 거쳐 전달. **불편**. 해결책:
- [[../state-management/state-management|전역 상태 관리]] (jotai, recoil, zustand).
- React Context.
- 컴포넌트 구조 재설계.

## 7. State 끌어올리기 (Lifting state up)

```tsx
// 두 자식이 같은 state 공유 필요?
function App() {
  const [count, setCount] = useState(0);    // 부모로 끌어올림
  return (
    <>
      <Display count={count} />
      <Controls setCount={setCount} />
    </>
  );
}

function Display({ count }) {
  return <p>{count}</p>;
}

function Controls({ setCount }) {
  return <button onClick={() => setCount(c => c + 1)}>+1</button>;
}
```

→ 형제 컴포넌트가 같은 state 사용 → **공통 부모로 끌어올림**.

## 8. Controlled vs Uncontrolled — Input 의 경우

```tsx
// Controlled — state 가 입력값을 controls
function ControlledInput() {
  const [value, setValue] = useState('');
  return (
    <input value={value} onChange={e => setValue(e.target.value)} />
  );
}

// Uncontrolled — DOM 이 자체적으로 보유, ref 로 read
function UncontrolledInput() {
  const ref = useRef<HTMLInputElement>(null);
  const handleSubmit = () => console.log(ref.current?.value);
  return <input ref={ref} />;
}
```

→ 대부분 **controlled**. [[forms-and-controlled-inputs]] 에서 자세히.

## 9. State 의 초기화 — Lazy initialization

```tsx
// 비싼 계산
function expensiveCompute() {
  console.log('compute!');
  return Array.from({length: 1000}).map((_, i) => i);
}

// ❌ 매 render 마다 expensiveCompute 호출
const [items, setItems] = useState(expensiveCompute());

// ✅ 함수 전달 — 첫 render 만 호출
const [items, setItems] = useState(() => expensiveCompute());
```

## 10. State 의 위치 — 어디 둘까?

1. **최대한 아래** — 한 컴포넌트만 쓰면 그 컴포넌트.
2. **공유 시 공통 조상**.
3. **전역 (어디서나)** — 전역 store (jotai/recoil/zustand).

오버 엔지니어링 함정: 처음부터 모든 state 를 전역으로. **로컬로 시작**.

## 11. Props 의 callback

```tsx
function App() {
  const [count, setCount] = useState(0);
  return <Child onAdd={() => setCount(c => c + 1)} />;
}

function Child({ onAdd }) {
  return <button onClick={onAdd}>+1</button>;
}
```

- 자식이 부모의 state 를 바꿔야 할 때 → **callback prop**.
- 자식 → 부모 방향의 통신.

## 12. 실전 예 — answer-fe 패턴

```bash
grep -r "useState" ~/masterway-dev/answer-fe/src/pages --include="*.tsx" -l | head -3
```

랜덤 1 개 열어 보면:
```tsx
const QuestionPage = () => {
  const [questions, setQuestions] = useState<Question[]>([]);
  const [page, setPage] = useState(0);
  const [isLoading, setIsLoading] = useState(false);
  // ...
};
```

→ 위 패턴 그대로.

## 13. 함정

1. **State 직접 mutate** — `items.push()`, `user.name = ...` ❌.
2. **useState 안 동기 가정** — `setX(...)` 직후 `x` 는 옛 값.
3. **함수형 update 안 씀** — `setCount(count + 1)` 두 번이 1 번만 증가.
4. **State 의 stale closure** — useEffect / setTimeout 안 옛 state. [[../pitfalls/stale-closures]] 참조.
5. **객체 state 의 깊은 update** — `setUser({...user, profile: {...user.profile, age: 31}})`. 깊으면 immer 권장.
6. **너무 많은 state** — 한 컴포넌트 useState 가 5+ → useReducer 또는 객체 state 통합.
7. **Render 안 setState** — 무한 loop. useEffect / event handler 안에서만.

## 14. 다음 단계

- [[hooks-essentials]] — useEffect / useRef / useMemo
- [[forms-and-controlled-inputs]] — controlled input 깊이

## 15. 관련

- [[core-concepts]]
- [[../state-management/state-management|전역 상태 관리]]
- [[../pitfalls/stale-closures|Stale closure 함정]]
