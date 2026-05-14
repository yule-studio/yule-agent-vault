---
title: "Conditional and List Rendering — 조건과 반복"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, jsx, conditional, list, key, beginner]
---

# Conditional and List Rendering

**[[core-concepts|↑ Core Concepts]]**

> JSX 안에서 if / for 못 씀. **삼항 / && / map 으로 대신**.

## 1. 조건부 렌더링 — 3 가지 패턴

### (A) 삼항연산자

```tsx
function Greeting({ isLoggedIn }) {
  return isLoggedIn
    ? <p>환영합니다!</p>
    : <p>로그인 해주세요.</p>;
}
```

### (B) `&&` 단축

```tsx
function Notification({ count }) {
  return (
    <div>
      <span>알림</span>
      {count > 0 && <Badge>{count}</Badge>}
    </div>
  );
}
```

→ `count > 0` 이 true 면 `<Badge>` 렌더.

### (C) early return

```tsx
function Profile({ user }) {
  if (!user) return <p>로딩 중...</p>;
  if (user.banned) return <p>차단된 사용자.</p>;

  return <h1>{user.name}</h1>;   // main case 만 main 위치에
}
```

→ guard 절로 nesting 줄이기.

## 2. `&&` 의 함정 — 0 / 빈 문자열

```tsx
const items = [];
return <div>{items.length && <List items={items} />}</div>;
// items.length === 0 → "0" 이 화면에 출력!
```

```tsx
// ✅ 명시적 boolean
{items.length > 0 && <List items={items} />}

// ✅ Boolean()
{Boolean(items.length) && <List items={items} />}

// ✅ 삼항으로 명시
{items.length > 0 ? <List items={items} /> : null}
```

→ React 는 `false`, `null`, `undefined`, `true` 를 안 그리지만 **`0` 은 그림** (string 변환).

## 3. List 렌더링 — `map`

```tsx
function UserList({ users }) {
  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

- `users.map(...)` 이 element 배열 return.
- React 가 배열을 풀어 그림.

## 4. `key` 의 중요성

```tsx
{items.map(item => <Row key={item.id} item={item} />)}
```

- `key` 는 React 가 **각 element 추적**.
- 같은 부모 안에서 **unique**.
- 형제끼리만 unique 면 OK (서로 다른 list 면 same key OK).

### 좋은 key

```tsx
// ✅ 안정적이고 unique
{items.map(item => <Row key={item.id} item={item} />)}
```

### 나쁜 key

```tsx
// ❌ index 사용 — list 가 reorder / 삽입 / 삭제 시 잘못된 동작
{items.map((item, i) => <Row key={i} item={item} />)}

// ❌ Math.random()
{items.map(item => <Row key={Math.random()} item={item} />)}
```

→ index 가 허용되는 경우: **list 가 절대 reorder/insert/delete 안 됨, 그리고 unique id 없음** (드뭄).

### key 가 잘못되면

```tsx
// 입력 form 의 list
{users.map((user, i) => (
  <div key={i}>
    <input defaultValue={user.name} />
  </div>
))}

// → user 한 명 삭제 시 key 가 index 라서 React 가
//   "user[1] 이 사라졌다" 가 아니라
//   "user[2] 가 user[1] 자리로 갔다" 로 해석.
//   → input 의 state 가 엉뚱한 user 에 붙음.
```

**불변 id 가 있다면 반드시 id 를 key 로**.

## 5. List + 조건

```tsx
// 활성 user 만
{users.filter(u => u.isActive).map(u => (
  <li key={u.id}>{u.name}</li>
))}

// 또는
{users.map(u => u.isActive && <li key={u.id}>{u.name}</li>)}
```

## 6. 빈 list — placeholder

```tsx
function UserList({ users }) {
  if (users.length === 0) {
    return <p>사용자가 없습니다.</p>;
  }
  return (
    <ul>
      {users.map(u => <li key={u.id}>{u.name}</li>)}
    </ul>
  );
}
```

## 7. Loading / Error / Empty / Data — 4 상태

```tsx
function UserList() {
  const { data: users, isLoading, error } = useQuery(['users'], fetchUsers);

  if (isLoading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;
  if (!users || users.length === 0) return <EmptyState />;

  return (
    <ul>
      {users.map(u => <li key={u.id}>{u.name}</li>)}
    </ul>
  );
}
```

→ 모든 list 컴포넌트의 표준 패턴. ([[../server-state/react-query]] 에서 자세히.)

## 8. switch / 매핑 패턴

```tsx
// ❌ 긴 if-else
function Icon({ type }) {
  if (type === 'home') return <HomeIcon />;
  if (type === 'user') return <UserIcon />;
  if (type === 'setting') return <SettingIcon />;
  return null;
}

// ✅ 객체 매핑
const ICONS = {
  home: HomeIcon,
  user: UserIcon,
  setting: SettingIcon,
};

function Icon({ type }) {
  const Component = ICONS[type];
  return Component ? <Component /> : null;
}
```

## 9. Fragment 안 list

```tsx
{items.map(item => (
  <React.Fragment key={item.id}>
    <dt>{item.label}</dt>
    <dd>{item.value}</dd>
  </React.Fragment>
))}
```

→ `<>` 단축 fragment 는 `key` 못 받음. `React.Fragment` 명시.

## 10. `null` / `undefined` 처리

```tsx
function Card({ user }) {
  return (
    <div>
      <h1>{user?.name ?? '이름 없음'}</h1>
      <p>{user?.bio || '소개 없음'}</p>
    </div>
  );
}
```

- `?.` (optional chaining) — null safe.
- `??` (nullish coalescing) — null/undefined 이면 default.
- `||` — falsy 값 (0, '', false 포함) 이면 default. 의도 다름!

## 11. answer-fe / job-answer-fe 패턴 보기

```bash
grep -rn ".map(" ~/masterway-dev/answer-fe/src/components --include="*.tsx" | head -5
```

전형적:
```tsx
{questions.map((q) => (
  <QuestionItem key={q.id} question={q} />
))}
```

조건부:
```tsx
{isLoading ? <Skeleton /> : <List data={data} />}
{!user && <LoginButton />}
{user?.isAdmin && <AdminPanel />}
```

## 12. 함정

1. **`&&` 와 `0`** — `{items.length && ...}` → `0` 출력.
2. **index 를 key 로** — reorder 시 state 엉킴.
3. **`Math.random()` key** — 매 render 마다 새 key → 전부 re-render.
4. **Key 누락** — console warning + reconciliation 성능 저하.
5. **`<>` 안 key** — `React.Fragment` 명시 필요.
6. **`||` 로 default** — `value || 'X'` 가 `value === 0` 일 때 'X' 가 됨. `??` 사용.

## 13. 다음 단계

- [[events-and-synthetic]] — 이벤트 처리
- [[forms-and-controlled-inputs]] — 입력 처리

## 14. 관련

- [[core-concepts]]
- [[../performance/memoization]] — list 의 re-render 최적화
- [[../pitfalls/re-render-loops]]
