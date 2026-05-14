---
title: "Events and Synthetic Events — onClick / onChange / 이벤트 처리"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, events, synthetic-events, beginner]
---

# Events and Synthetic Events

**[[core-concepts|↑ Core Concepts]]**

> React 의 이벤트 = `onClick={handler}` 형식. 일반 DOM event 와 비슷하지만 React 가 wrapping (SyntheticEvent).

## 1. 기본 — onClick

```tsx
function Button() {
  const handleClick = () => {
    console.log('clicked!');
  };

  return <button onClick={handleClick}>Click</button>;
}
```

- `onClick={handleClick}` — 함수 **참조** 전달.
- `onClick={handleClick()}` ❌ — 즉시 호출돼 return 값이 전달됨.

## 2. Inline 함수 vs 외부 함수

```tsx
// inline (간단할 때)
<button onClick={() => setCount(c => c + 1)}>+</button>

// 외부 함수 (복잡 / 재사용)
const handleClick = () => {
  setCount(c => c + 1);
  setLog(l => [...l, 'inc']);
};
<button onClick={handleClick}>+</button>
```

## 3. 이벤트 객체 (SyntheticEvent)

```tsx
const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => {
  console.log(e.target);            // 클릭 대상 element
  console.log(e.currentTarget);     // 핸들러 붙은 element
  e.preventDefault();               // 기본 동작 막기
  e.stopPropagation();              // 상위 전파 막기
};
```

→ React 의 `e` 는 SyntheticEvent. 일반 native event 처럼 동작.

## 4. 흔한 이벤트들

| Prop | 트리거 |
| --- | --- |
| `onClick` | 클릭 |
| `onDoubleClick` | 더블 클릭 |
| `onMouseEnter` / `onMouseLeave` | 마우스 hover |
| `onChange` | input/select/textarea 값 변경 |
| `onInput` | input 값 입력 (matter: onChange 와 비슷) |
| `onSubmit` | form 제출 |
| `onFocus` / `onBlur` | input focus / 잃음 |
| `onKeyDown` / `onKeyUp` / `onKeyPress` | 키보드 |
| `onScroll` | 스크롤 |
| `onLoad` / `onError` | img / iframe |

→ HTML 의 `onclick` 가 camelCase `onClick`.

## 5. onChange — input 변경

```tsx
function NameInput() {
  const [name, setName] = useState('');

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setName(e.target.value);
  };

  return <input value={name} onChange={handleChange} />;
}
```

### `e.target.value` 의 type

```tsx
// input
e.target.value         // string
e.target.checked       // boolean (checkbox / radio)

// select
e.target.value         // string

// file input
e.target.files         // FileList | null
```

## 6. onSubmit — form

```tsx
function LoginForm() {
  const [email, setEmail] = useState('');

  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();   // 기본 페이지 reload 막기 (필수!)
    console.log('submit', email);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input value={email} onChange={e => setEmail(e.target.value)} />
      <button type="submit">로그인</button>
    </form>
  );
}
```

→ `e.preventDefault()` 빠뜨리면 페이지 reload.

## 7. onKeyDown — 키보드

```tsx
const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => {
  if (e.key === 'Enter') {
    submit();
  }
  if (e.key === 'Escape') {
    cancel();
  }
};
<input onKeyDown={handleKeyDown} />
```

- `e.key` — 키 이름 ('a', 'Enter', 'Escape', 'ArrowUp').
- `e.code` — 물리 키 ('KeyA').
- `e.shiftKey`, `e.ctrlKey`, `e.metaKey`, `e.altKey` — modifier.

## 8. Argument 전달 — 추가 인자

```tsx
function List({ items }) {
  const handleDelete = (id: number) => {
    console.log('delete', id);
  };

  return (
    <ul>
      {items.map(item => (
        <li key={item.id}>
          {item.name}
          {/* ✅ inline arrow */}
          <button onClick={() => handleDelete(item.id)}>X</button>

          {/* ❌ 즉시 호출 — render 시 바로 실행 */}
          <button onClick={handleDelete(item.id)}>X</button>
        </li>
      ))}
    </ul>
  );
}
```

→ 추가 인자 필요할 때 **arrow 로 wrap**.

## 9. 이벤트 위임 — 부모에서 한 번에

```tsx
function List({ items }) {
  const handleClick = (e: React.MouseEvent<HTMLUListElement>) => {
    const target = e.target as HTMLElement;
    const id = target.dataset.id;
    if (id) console.log('clicked', id);
  };

  return (
    <ul onClick={handleClick}>
      {items.map(item => (
        <li key={item.id} data-id={item.id}>{item.name}</li>
      ))}
    </ul>
  );
}
```

→ React 의 이벤트는 자동 위임 (root 에서 처리). 일반적으로 직접 element 에 붙여도 성능 OK.

## 10. preventDefault / stopPropagation

```tsx
// preventDefault — 기본 동작 막기
<a href="/path" onClick={e => {
  e.preventDefault();
  // SPA 라우팅
}}>링크</a>

// stopPropagation — 부모로 안 올라가게
<div onClick={() => console.log('outer')}>
  <button onClick={e => {
    e.stopPropagation();   // outer 안 실행
    console.log('inner');
  }}>Click</button>
</div>
```

## 11. Async handler

```tsx
const handleClick = async () => {
  setLoading(true);
  try {
    await api.save(data);
    setSuccess(true);
  } catch (err) {
    setError(err);
  } finally {
    setLoading(false);
  }
};

<button onClick={handleClick} disabled={loading}>저장</button>
```

→ async 함수 직접 OK. 단 React onSubmit 의 `e.preventDefault` 는 첫 `await` 전에 호출.

## 12. SyntheticEvent 의 pooling (React 17 미만 함정)

```tsx
// React 16: e 가 pool 에 반환됨
const handleChange = e => {
  setTimeout(() => console.log(e.target.value), 1000);   // null
};

// 해결 — e.persist() 또는 값 미리 추출
const handleChange = e => {
  const value = e.target.value;
  setTimeout(() => console.log(value), 1000);
};
```

→ **React 17 부터 pooling 폐지**. 신경 안 써도 됨.

## 13. TypeScript 이벤트 타입 표

| Element | Event |
| --- | --- |
| `<button onClick>` | `React.MouseEvent<HTMLButtonElement>` |
| `<input onChange>` | `React.ChangeEvent<HTMLInputElement>` |
| `<textarea onChange>` | `React.ChangeEvent<HTMLTextAreaElement>` |
| `<select onChange>` | `React.ChangeEvent<HTMLSelectElement>` |
| `<form onSubmit>` | `React.FormEvent<HTMLFormElement>` |
| `<input onKeyDown>` | `React.KeyboardEvent<HTMLInputElement>` |
| `<input onFocus>` | `React.FocusEvent<HTMLInputElement>` |

→ inline 함수면 TS 가 자동 추론. 외부 함수면 명시.

## 14. 함정

1. **`onClick={handler()}`** — 즉시 호출. `onClick={handler}` 또는 `onClick={() => handler()}`.
2. **form 의 `e.preventDefault()` 누락** — 페이지 reload.
3. **`onChange` 의 `e.target.value`** — checkbox 는 `.checked` 가 맞음.
4. **이벤트 객체 비동기 사용 (React 16)** — React 17+ 는 OK.
5. **map 에서 handler inline 생성** — 매 render 마다 새 함수. `React.memo` 자식이면 매번 re-render. useCallback / 이벤트 위임.
6. **`onKeyPress` deprecated** — `onKeyDown` 사용.
7. **부모 자식의 동시 onClick** — 위로 전파. `e.stopPropagation()` 으로 제어.

## 15. 다음 단계

- [[forms-and-controlled-inputs]] — form 처리 깊이
- [[../performance/memoization]]

## 16. 관련

- [[core-concepts]]
- [[../forms/react-hook-form]]
