---
title: "Component Types — Props / Children / forwardRef"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, typescript, component-types, props, forwardref]
---

# Component Types

**[[typescript|↑ TypeScript Hub]]**

> React 컴포넌트의 TypeScript 패턴.

## 1. 기본 — Props 타입

```tsx
interface ButtonProps {
  label: string;
  onClick: () => void;
}

function Button({ label, onClick }: ButtonProps) {
  return <button onClick={onClick}>{label}</button>;
}
```

## 2. children 의 타입

```tsx
import { ReactNode, ReactElement, PropsWithChildren } from 'react';

// 방법 1
interface CardProps {
  children: ReactNode;
}

// 방법 2 — PropsWithChildren helper
type CardProps = PropsWithChildren<{ title: string }>;
// → { title: string; children?: ReactNode }

// 방법 3 — 단일 element 만
interface ModalProps {
  children: ReactElement;
}
```

### ReactNode vs ReactElement
| | 포함 |
| --- | --- |
| `ReactNode` | element, string, number, null, undefined, boolean, array |
| `ReactElement` | JSX 한 개 (`<X />`) 만 |

→ 대부분 `ReactNode`.

## 3. Optional / default

```tsx
interface ButtonProps {
  label: string;
  variant?: 'primary' | 'secondary';      // ?
  disabled?: boolean;
}

function Button({ label, variant = 'primary', disabled = false }: ButtonProps) {
  // ...
}
```

## 4. 함수 prop

```tsx
interface FormProps {
  onSubmit: (data: { email: string; password: string }) => void;
  onCancel?: () => void;
  onChange?: (field: string, value: string) => void;
}
```

## 5. extending HTMLElement props

```tsx
// 일반 button 의 모든 prop + label
interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  label: string;
  variant?: 'primary' | 'secondary';
}

function Button({ label, variant, ...rest }: ButtonProps) {
  return <button {...rest}>{label}</button>;
}

// 사용
<Button label="제출" onClick={...} disabled type="submit" />
```

→ HTML 속성 + 커스텀 prop 합치는 핵심 패턴.

### 자주 쓰는 base
```ts
React.ButtonHTMLAttributes<HTMLButtonElement>
React.InputHTMLAttributes<HTMLInputElement>
React.HTMLAttributes<HTMLDivElement>
React.AnchorHTMLAttributes<HTMLAnchorElement>
React.TextareaHTMLAttributes<HTMLTextAreaElement>
React.SelectHTMLAttributes<HTMLSelectElement>
```

## 6. FC (FunctionComponent) — 쓸까 말까

```tsx
// FC 사용
const Button: React.FC<ButtonProps> = ({ label }) => <button>{label}</button>;

// 또는 그냥 함수
function Button({ label }: ButtonProps) { ... }
const Button = ({ label }: ButtonProps) => { ... };
```

→ **공식 권장은 그냥 함수**. `FC` 가 옛날에 implicit children 을 넣어 문제 → React 18+ 에서 fix 됐지만 굳이 안 씀.

`masterway-dev/answer-fe` 도 `FC` 없이 사용.

## 7. forwardRef

```tsx
import { forwardRef } from 'react';

interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string;
}

const Input = forwardRef<HTMLInputElement, InputProps>(({ label, ...rest }, ref) => {
  return (
    <label>
      {label}
      <input ref={ref} {...rest} />
    </label>
  );
});

Input.displayName = 'Input';

// 사용
const ref = useRef<HTMLInputElement>(null);
<Input ref={ref} label="이름" />
```

→ 부모가 자식 element 에 직접 ref 접근 필요할 때. UI 라이브러리에서 자주.

## 8. Generic 컴포넌트

```tsx
interface SelectProps<T> {
  options: T[];
  getLabel: (item: T) => string;
  onSelect: (item: T) => void;
}

function Select<T>({ options, getLabel, onSelect }: SelectProps<T>) {
  return (
    <ul>
      {options.map((opt, i) => (
        <li key={i} onClick={() => onSelect(opt)}>
          {getLabel(opt)}
        </li>
      ))}
    </ul>
  );
}

// 사용
<Select<User>
  options={users}
  getLabel={u => u.name}
  onSelect={u => console.log(u.id)}
/>
```

→ 어떤 타입의 list 든 받는 재사용 컴포넌트.

## 9. polymorphic 컴포넌트 — `as` prop

```tsx
// MUI / Mantine / Chakra 같은 lib 패턴
type BoxProps<C extends React.ElementType> = {
  as?: C;
  children: ReactNode;
} & React.ComponentPropsWithoutRef<C>;

function Box<C extends React.ElementType = 'div'>({
  as,
  children,
  ...rest
}: BoxProps<C>) {
  const Component = as || 'div';
  return <Component {...rest}>{children}</Component>;
}

// 사용
<Box>Default div</Box>
<Box as="a" href="https://...">Link</Box>
<Box as="button" onClick={...}>Button</Box>
```

→ 어려운 패턴. UI 라이브러리 개발자 외에는 직접 만들 일 적음.

## 10. styled-components 컴포넌트 타입

```tsx
import styled from 'styled-components';

interface BoxProps {
  $padding?: number;       // $ prefix = transient prop (DOM 으로 안 내려감)
  $bg?: string;
}

const Box = styled.div<BoxProps>`
  padding: ${p => p.$padding ?? 16}px;
  background: ${p => p.$bg ?? 'white'};
`;

<Box $padding={20} $bg="lightgray">...</Box>
```

→ [[../styling/styled-components]] 에서 자세히.

## 11. discriminated union — variant 별 prop

```tsx
type AlertProps =
  | { type: 'success'; message: string }
  | { type: 'error'; message: string; retry: () => void }
  | { type: 'info'; message: string };

function Alert(props: AlertProps) {
  if (props.type === 'error') {
    return <div>{props.message} <button onClick={props.retry}>재시도</button></div>;
  }
  return <div>{props.message}</div>;
}

// 사용
<Alert type="success" message="저장 완료" />
<Alert type="error" message="실패" retry={() => ...} />
// <Alert type="error" message="실패" />   // ❌ retry 누락
```

→ type 별로 필요한 prop 강제.

## 12. ComponentProps — 다른 컴포넌트의 prop 추출

```tsx
import { ComponentProps } from 'react';
import { Button } from './Button';

// Button 컴포넌트의 prop 그대로 + 추가
interface MyButtonProps extends ComponentProps<typeof Button> {
  isLoading?: boolean;
}
```

## 13. answer-fe / job-answer-fe 의 흔한 패턴

```tsx
// answer-fe — interface + 함수 컴포넌트
interface QuestionItemProps {
  question: Question;
  onClick: (id: number) => void;
}

const QuestionItem = ({ question, onClick }: QuestionItemProps) => {
  return <li onClick={() => onClick(question.id)}>{question.title}</li>;
};
```

```tsx
// styled-components 와 결합
interface ButtonProps {
  variant?: 'primary' | 'ghost';
  size?: 'sm' | 'md' | 'lg';
}

const Button = styled.button<ButtonProps>`
  padding: ${p => p.size === 'sm' ? '4px 8px' : '8px 16px'};
  background: ${p => p.variant === 'primary' ? '#000' : 'transparent'};
`;
```

## 14. 함정

1. **`children: any`** — `ReactNode` 사용.
2. **`React.FC` 사용** — 굳이 안 씀.
3. **`onClick: Function`** — `() => void` 등 구체적.
4. **`useRef<HTMLInputElement>()`** — 초기값 누락. `useRef<HTMLInputElement>(null)`.
5. **HTML 속성 누락** — `extends ButtonHTMLAttributes<...>` 로 합쳐 전달 가능하게.
6. **Generic 컴포넌트의 inline arrow** — `<T,>` 으로 trailing comma (JSX 충돌).
   ```tsx
   const X = <T,>(x: T) => x;     // ✅
   const X = <T>(x: T) => x;       // ❌ JSX 로 해석
   ```

## 15. 다음 단계

- [[../core-concepts/hooks-essentials]]
- [[../styling/styled-components]] — styled-components 타입

## 16. 관련

- [[typescript]]
- [[../core-concepts/props-and-state]]
