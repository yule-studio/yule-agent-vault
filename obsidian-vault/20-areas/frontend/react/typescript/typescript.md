---
title: "TypeScript in React — 타입 시스템 길잡이"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, typescript, hub, beginner]
---

# TypeScript in React

**[[../react|↑ React Hub]]**

> JS 에 타입을 더한 언어. React 와 짝꿍. `masterway-dev` 의 모든 -fe 가 TS.

## 1. 왜 TS?

- prop 의 타입을 IDE 가 알려줌 → 실수 즉시 발견.
- 자동 완성 (member, method 이름).
- refactor 안전성 — 이름 바꾸면 모든 사용처 자동 변경.
- 팀 코드의 self-documenting.

## 2. 기본 — 변수 / 함수 타입

```ts
const name: string = '유철';
const age: number = 30;
const isActive: boolean = true;
const items: string[] = ['a', 'b'];
const user: { id: number; name: string } = { id: 1, name: '유철' };

function add(a: number, b: number): number {
  return a + b;
}

const greet = (name: string): string => `안녕, ${name}`;
```

## 3. 객체 타입 — `interface` vs `type`

```ts
// interface
interface User {
  id: number;
  name: string;
  email?: string;     // optional
}

// type
type User = {
  id: number;
  name: string;
  email?: string;
};
```

- 둘 다 비슷. 객체 → `interface`, 합성 / union → `type` 권장.
- `interface` 는 extends / declaration merge 가능.

```ts
interface Animal { name: string; }
interface Dog extends Animal { bark: () => void; }

type Result = { ok: true; data: string } | { ok: false; error: string };  // union
```

## 4. Component Props 타입

```tsx
// 방법 1 — inline
function Greeting({ name, age }: { name: string; age: number }) {
  return <p>{name}, {age}</p>;
}

// 방법 2 — interface
interface GreetingProps {
  name: string;
  age: number;
}
function Greeting({ name, age }: GreetingProps) { ... }

// 방법 3 — type
type GreetingProps = {
  name: string;
  age: number;
};
const Greeting = ({ name, age }: GreetingProps) => { ... };
```

→ 자세히 [[component-types]].

## 5. useState 타입

```tsx
const [count, setCount] = useState(0);           // 자동 추론 number

const [user, setUser] = useState<User | null>(null);    // 명시 (초기값 null)

const [items, setItems] = useState<string[]>([]);       // 빈 배열도 명시
```

## 6. 이벤트 타입

```tsx
const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => { ... };
const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => { ... };
const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => { ... };
```

→ 자세히 [[../core-concepts/events-and-synthetic]].

## 7. ref 타입

```tsx
const ref = useRef<HTMLInputElement>(null);
ref.current?.focus();
```

## 8. children prop

```tsx
import { ReactNode } from 'react';

interface CardProps {
  title: string;
  children: ReactNode;
}

function Card({ title, children }: CardProps) {
  return <div><h1>{title}</h1>{children}</div>;
}
```

## 9. union / literal 타입

```ts
type Status = 'idle' | 'loading' | 'success' | 'error';
const [status, setStatus] = useState<Status>('idle');

if (status === 'loading') { ... }
if (status === 'error') { ... }
```

→ 잘못된 string 넣으면 TS 가 즉시 잡음.

## 10. Generic — 재사용 타입

```ts
function identity<T>(value: T): T { return value; }

const a = identity<number>(5);        // T = number
const b = identity('hello');           // T = string (자동 추론)

// API 응답
interface ApiResponse<T> {
  data: T;
  error?: string;
}
const userResp: ApiResponse<User> = { data: { id: 1, name: '유철' } };
```

## 11. 자주 쓰는 utility 타입

```ts
type User = { id: number; name: string; email: string };

Partial<User>            // 모든 field optional
Required<User>           // 모든 field 필수
Readonly<User>           // 모든 field readonly
Pick<User, 'id'>         // { id: number }
Omit<User, 'email'>      // { id, name } 만
Record<string, number>   // { [key: string]: number }
ReturnType<typeof fn>    // 함수 return 타입
Parameters<typeof fn>    // 함수 인자 타입 (튜플)
```

→ 매우 자주 사용. 숙지.

## 12. Type assertion (캐스팅)

```ts
const el = document.getElementById('app') as HTMLDivElement;
const user = data as User;   // 위험 — 검증 없이 단언

// 안전한 narrow
if (typeof x === 'string') { /* x: string */ }
if (Array.isArray(x)) { /* x: any[] */ }
if (x instanceof Error) { /* x: Error */ }
```

→ `as` 는 신중히. 가능하면 type narrowing.

## 13. unknown vs any

```ts
let x: any = 'hello';      // 아무거나, TS 검사 안 함 — 위험
x.foo.bar.baz;             // 통과 (런타임 에러)

let y: unknown = 'hello';  // 아무거나, 하지만 좁히기 전엔 사용 불가
y.foo;                     // ❌ Error
if (typeof y === 'string') y.toUpperCase();  // ✅
```

→ **`any` 는 가능한 피하기**. `unknown` 으로 시작 → narrow.

## 14. API 응답 타입화

```ts
interface User { id: number; name: string; }

async function fetchUser(id: number): Promise<User> {
  const r = await fetch(`/api/user/${id}`);
  return r.json();    // 실제로는 unknown. 신뢰 또는 zod 검증.
}
```

→ 안전 보장은 zod / io-ts 같은 runtime 검증.

## 15. 자기 프로젝트의 tsconfig

```bash
cat ~/masterway-dev/answer-fe/tsconfig.json
```

자주 보이는 옵션:
- `"strict": true` — 모든 strict 옵션 켜기 (권장).
- `"jsx": "react-jsx"` — React 17+ 의 새 JSX transform.
- `"paths": { "@/*": ["./src/*"] }` — alias.

## 16. 학습 순서

1. **[[component-types]]** — 컴포넌트 prop / 함수 컴포넌트 / forwardRef.
2. **기본 type / interface 익히기**.
3. **utility 타입 (`Partial`, `Pick`, `Omit`) 사용**.
4. **generic** 으로 reusable 함수 / hook 작성.
5. **타입 narrowing** 으로 unknown 안전 처리.

## 17. 함정

1. **`any` 남용** — TS 효용 사라짐.
2. **`as` 강제 캐스팅** — 런타임 에러로 이어짐.
3. **`// @ts-ignore`** — 마지막 수단. 이유 주석 필수.
4. **`Function` 타입** — 너무 광범. `(x: number) => string` 처럼 구체.
5. **`Object`, `{}`** — `Record<string, unknown>` 또는 더 구체적.

## 18. 외부 자료

- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html)
- [React + TypeScript Cheatsheet](https://react-typescript-cheatsheet.netlify.app/)
- [Total TypeScript](https://www.totaltypescript.com/) — Matt Pocock 무료 튜토리얼.

## 19. 관련

- [[component-types]]
- [[../core-concepts/core-concepts]]
- [[../configuration|tsconfig.json]]
