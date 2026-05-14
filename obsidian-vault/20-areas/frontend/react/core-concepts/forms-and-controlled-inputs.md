---
title: "Forms and Controlled Inputs — 입력 다루기"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, forms, controlled-input, validation, beginner]
---

# Forms and Controlled Inputs

**[[core-concepts|↑ Core Concepts]]**

> 입력 = state. `value + onChange` 짝.

## 1. Controlled Input — 기본

```tsx
function EmailInput() {
  const [email, setEmail] = useState('');

  return (
    <input
      type="email"
      value={email}                              // ① state 가 값
      onChange={e => setEmail(e.target.value)}   // ② 변경 시 state 업데이트
    />
  );
}
```

→ **React state = 진실의 원천**. 사용자 입력 = state 의 거울.

## 2. 여러 입력 — 단일 객체

```tsx
function LoginForm() {
  const [form, setForm] = useState({ email: '', password: '' });

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setForm(prev => ({ ...prev, [e.target.name]: e.target.value }));
  };

  return (
    <form>
      <input name="email" value={form.email} onChange={handleChange} />
      <input name="password" type="password" value={form.password} onChange={handleChange} />
    </form>
  );
}
```

→ `name` 속성으로 어느 field 인지 식별. `[e.target.name]` = computed property.

## 3. checkbox / radio

```tsx
function AgreeBox() {
  const [agreed, setAgreed] = useState(false);

  return (
    <input
      type="checkbox"
      checked={agreed}                           // value 아닌 checked
      onChange={e => setAgreed(e.target.checked)}
    />
  );
}
```

```tsx
function Gender() {
  const [gender, setGender] = useState('');

  return (
    <>
      <label>
        <input type="radio" name="gender" value="m"
          checked={gender === 'm'} onChange={e => setGender(e.target.value)} />
        남
      </label>
      <label>
        <input type="radio" name="gender" value="f"
          checked={gender === 'f'} onChange={e => setGender(e.target.value)} />
        여
      </label>
    </>
  );
}
```

## 4. select

```tsx
function FruitSelect() {
  const [fruit, setFruit] = useState('apple');

  return (
    <select value={fruit} onChange={e => setFruit(e.target.value)}>
      <option value="apple">사과</option>
      <option value="banana">바나나</option>
      <option value="grape">포도</option>
    </select>
  );
}
```

→ React 는 `selected` 속성 안 씀. `<select value=...>` 로 통일.

### Multi-select

```tsx
const [fruits, setFruits] = useState<string[]>([]);

<select
  multiple
  value={fruits}
  onChange={e => {
    const selected = Array.from(e.target.selectedOptions, o => o.value);
    setFruits(selected);
  }}
>
  ...
</select>
```

## 5. textarea

```tsx
<textarea
  value={content}
  onChange={e => setContent(e.target.value)}
  rows={5}
/>
```

→ HTML 의 `<textarea>본문</textarea>` 와 달리 React 는 `value` prop.

## 6. file input

```tsx
function FileUpload() {
  const [file, setFile] = useState<File | null>(null);

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setFile(e.target.files?.[0] ?? null);
  };

  return (
    <>
      <input type="file" onChange={handleChange} accept="image/*" />
      {file && <p>{file.name} ({Math.round(file.size / 1024)} KB)</p>}
    </>
  );
}
```

→ file input 은 보안상 **uncontrolled** (value 못 set). value prop 사용 X.

### 미리보기 + 업로드

```tsx
const handleChange = async (e) => {
  const f = e.target.files?.[0];
  if (!f) return;

  // 미리보기
  const previewUrl = URL.createObjectURL(f);
  setPreview(previewUrl);

  // 업로드 (FormData)
  const fd = new FormData();
  fd.append('file', f);
  await fetch('/upload', { method: 'POST', body: fd });
};
```

## 7. submit 처리

```tsx
function ContactForm() {
  const [form, setForm] = useState({ name: '', message: '' });

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();   // 페이지 reload 막기
    await api.contact(form);
    setForm({ name: '', message: '' });
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="name" value={form.name} onChange={...} />
      <textarea name="message" value={form.message} onChange={...} />
      <button type="submit">보내기</button>
    </form>
  );
}
```

→ `<form onSubmit>` + `<button type="submit">` 패턴. Enter 키도 submit.

## 8. 단순 validation (수동)

```tsx
function SignupForm() {
  const [email, setEmail] = useState('');
  const [errors, setErrors] = useState<Record<string, string>>({});

  const validate = () => {
    const next: Record<string, string> = {};
    if (!email) next.email = '이메일 필수';
    else if (!email.includes('@')) next.email = '이메일 형식';
    setErrors(next);
    return Object.keys(next).length === 0;
  };

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    if (!validate()) return;
    api.signup({ email });
  };

  return (
    <form onSubmit={handleSubmit}>
      <input value={email} onChange={e => setEmail(e.target.value)} />
      {errors.email && <p style={{ color: 'red' }}>{errors.email}</p>}
      <button type="submit">가입</button>
    </form>
  );
}
```

→ 실전에서는 [[../forms/react-hook-form|react-hook-form]] + zod 권장.

## 9. Controlled vs Uncontrolled

### Controlled
```tsx
<input value={x} onChange={...} />   // state 가 진실
```
- 변경 즉시 read.
- 조건부 enable / 즉시 validation 쉬움.
- 매 키 입력마다 re-render.

### Uncontrolled
```tsx
const ref = useRef<HTMLInputElement>(null);
<input defaultValue="..." ref={ref} />
// submit 시 ref.current.value 로 read
```
- 단순 form / 큰 form 의 성능.
- 외부 lib (jQuery 류) 와 통합.

→ 대부분 controlled. 큰 form 의 성능 이슈 시 react-hook-form (내부 uncontrolled).

## 10. 입력 즉시 검증 / debounce

```tsx
import { useEffect, useState } from 'react';

function Search() {
  const [query, setQuery] = useState('');
  const [debounced, setDebounced] = useState(query);

  useEffect(() => {
    const t = setTimeout(() => setDebounced(query), 300);
    return () => clearTimeout(t);
  }, [query]);

  useEffect(() => {
    if (debounced) {
      fetch(`/search?q=${debounced}`).then(...);
    }
  }, [debounced]);

  return <input value={query} onChange={e => setQuery(e.target.value)} />;
}
```

→ 매 키 입력마다 API 호출 ❌. 300ms debounce.

## 11. 실전 — answer-fe 패턴

`masterway-dev/answer-fe` 는 [[../forms/react-hook-form|react-hook-form]] + zod 표준:

```tsx
const { register, handleSubmit, formState: { errors } } = useForm<FormType>({
  resolver: zodResolver(schema),
});

<form onSubmit={handleSubmit(onValid)}>
  <input {...register('email')} />
  {errors.email && <p>{errors.email.message}</p>}
  <button type="submit">제출</button>
</form>
```

→ 한 form 의 모든 state / validation 위임. [[../forms/react-hook-form]] 에서 상세.

## 12. 한국어 IME 입력 (Composition)

```tsx
function KoreanInput() {
  const [value, setValue] = useState('');
  const [composing, setComposing] = useState(false);

  return (
    <input
      value={value}
      onChange={e => setValue(e.target.value)}
      onCompositionStart={() => setComposing(true)}
      onCompositionEnd={() => setComposing(false)}
      onKeyDown={e => {
        if (composing) return;        // 조합 중 enter 무시
        if (e.key === 'Enter') submit();
      }}
    />
  );
}
```

→ 한글 입력 시 Enter 가 의도치 않게 submit 되는 문제 해결.

## 13. 함정

1. **`value` 없이 `onChange` 만** — uncontrolled 처럼 동작.
2. **`value={undefined}`** — controlled / uncontrolled 경고. `value={x ?? ''}` 또는 처음부터 빈 문자열.
3. **checkbox 의 `value` vs `checked`** — 반드시 `checked`.
4. **form submit 의 `e.preventDefault()` 누락** — 페이지 reload.
5. **한국어 입력 시 Enter** — composition 이벤트 처리.
6. **매 키 입력 API 호출** — debounce 필수.
7. **file input 의 reset** — `value={...}` 못 함. `ref.current.value = ''` 또는 key 변경으로 remount.

## 14. 다음 단계

- [[../forms/react-hook-form]] — 실전 form 처리
- [[../forms/forms]] — Form hub

## 15. 관련

- [[core-concepts]]
- [[events-and-synthetic]]
- [[../forms/forms]]
