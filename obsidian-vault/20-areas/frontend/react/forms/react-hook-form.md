---
title: "react-hook-form — 실전 폼 처리"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, forms, react-hook-form, zod]
---

# react-hook-form

**[[forms|↑ Forms Hub]]**

> uncontrolled 기반 빠른 폼. + zod 로 type-safe validation.

## 1. 설치

```bash
yarn add react-hook-form
yarn add zod @hookform/resolvers      # zod 통합
```

## 2. 기본 — register / handleSubmit / errors

```tsx
import { useForm } from 'react-hook-form';

interface Inputs {
  email: string;
  password: string;
}

function LoginForm() {
  const { register, handleSubmit, formState: { errors } } = useForm<Inputs>();

  const onSubmit = (data: Inputs) => {
    console.log(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('email', { required: '필수', pattern: /\S+@\S+/ })} />
      {errors.email && <p>{errors.email.message ?? '이메일 형식'}</p>}

      <input
        type="password"
        {...register('password', { required: '필수', minLength: { value: 8, message: '8자 이상' } })}
      />
      {errors.password && <p>{errors.password.message}</p>}

      <button type="submit">로그인</button>
    </form>
  );
}
```

→ `register` 가 input 에 ref + onChange + onBlur 자동 연결. 매 키 입력에 re-render X.

## 3. zod 와 통합 — `zodResolver`

```tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const schema = z.object({
  email: z.string().email('이메일 형식'),
  password: z.string().min(8, '8자 이상'),
});

type FormData = z.infer<typeof schema>;

function LoginForm() {
  const { register, handleSubmit, formState: { errors, isSubmitting } } = useForm<FormData>({
    resolver: zodResolver(schema),
  });

  return (
    <form onSubmit={handleSubmit(async (data) => await api.login(data))}>
      <input {...register('email')} />
      {errors.email && <p>{errors.email.message}</p>}
      ...
      <button disabled={isSubmitting}>로그인</button>
    </form>
  );
}
```

→ zod schema 가 곧 TypeScript type 이 됨 (`z.infer`).

## 4. formState 의 값들

```ts
const { formState } = useForm();

formState.errors                  // field 별 에러
formState.isSubmitting            // 제출 중
formState.isSubmitSuccessful      // 마지막 제출 성공
formState.isDirty                 // 1 번이라도 수정
formState.dirtyFields              // 어느 field 가 수정됐나
formState.touchedFields            // 어느 field 가 focus 됐었나
formState.isValid                 // 전체 validation 통과
formState.isLoading                // async default value loading
```

## 5. Controller — controlled component (MUI / antd) 와

```tsx
import { Controller, useForm } from 'react-hook-form';
import { TextField } from '@mui/material';

const { control, handleSubmit } = useForm<Inputs>();

<Controller
  control={control}
  name="email"
  rules={{ required: '필수' }}
  render={({ field, fieldState }) => (
    <TextField
      {...field}
      label="이메일"
      error={!!fieldState.error}
      helperText={fieldState.error?.message}
    />
  )}
/>
```

→ `register` 는 native input 만. MUI / antd 같은 controlled lib 는 Controller.

## 6. default values

```tsx
const { register } = useForm<Inputs>({
  defaultValues: {
    email: 'preset@example.com',
    role: 'USER',
  },
});
```

### async default — reset 으로
```tsx
const { reset } = useForm<Inputs>();
const { data: user } = useQuery(['user', id], fetchUser);

useEffect(() => {
  if (user) reset({ name: user.name, email: user.email });
}, [user, reset]);
```

## 7. watch — 다른 field 의 값에 따라 동작

```tsx
const { watch } = useForm<Inputs>();

const password = watch('password');
const accountType = watch('accountType');

return (
  <>
    <input {...register('password')} />
    <p>비밀번호 길이: {password?.length ?? 0}</p>

    {accountType === 'business' && <input {...register('companyName')} />}
  </>
);
```

→ `watch` 가 호출되면 그 field 의 변화에 컴포넌트가 re-render. 너무 잦으면 성능.

## 8. setValue / getValues

```tsx
const { setValue, getValues, trigger } = useForm<Inputs>();

setValue('email', 'new@example.com');
const all = getValues();
const just = getValues('email');

trigger('email');           // 수동 validation
trigger();                  // 전체
```

## 9. setError — 서버 에러 표시

```tsx
const { setError } = useForm<Inputs>();

const onSubmit = async (data: Inputs) => {
  try {
    await api.signup(data);
  } catch (e: any) {
    if (e.response?.data?.field === 'email') {
      setError('email', { message: e.response.data.message });
    } else {
      setError('root.serverError', { message: '서버 오류' });
    }
  }
};

{errors.root?.serverError && <p>{errors.root.serverError.message}</p>}
```

## 10. dynamic fields — useFieldArray

```tsx
import { useFieldArray, useForm } from 'react-hook-form';

interface Form {
  todos: { text: string }[];
}

function TodoForm() {
  const { control, register, handleSubmit } = useForm<Form>({
    defaultValues: { todos: [{ text: '' }] },
  });

  const { fields, append, remove } = useFieldArray({ control, name: 'todos' });

  return (
    <form onSubmit={handleSubmit(console.log)}>
      {fields.map((field, i) => (
        <div key={field.id}>
          <input {...register(`todos.${i}.text`)} />
          <button type="button" onClick={() => remove(i)}>X</button>
        </div>
      ))}
      <button type="button" onClick={() => append({ text: '' })}>추가</button>
      <button type="submit">저장</button>
    </form>
  );
}
```

→ list 형 필드의 표준.

## 11. mode — validation 시점

```tsx
useForm({ mode: 'onSubmit' });   // 기본 — submit 시
useForm({ mode: 'onBlur' });     // blur 시
useForm({ mode: 'onChange' });   // 입력마다 (잦은 re-render)
useForm({ mode: 'all' });        // onChange + onBlur
```

## 12. nested object / array — dot path

```ts
register('user.email')                   // { user: { email: ... } }
register('addresses.0.street')           // { addresses: [{ street: ... }] }
```

```ts
const schema = z.object({
  user: z.object({
    email: z.string().email(),
    age: z.number(),
  }),
  addresses: z.array(z.object({ street: z.string() })),
});
```

## 13. 실전 — answer-fe / job-answer-fe 패턴

```tsx
// schemas/login.ts
export const loginSchema = z.object({
  email: z.string().email('이메일 형식'),
  password: z.string().min(8, '8자 이상'),
});
export type LoginForm = z.infer<typeof loginSchema>;

// hooks/useLogin.ts
export const useLoginForm = () => {
  return useForm<LoginForm>({ resolver: zodResolver(loginSchema) });
};

// LoginPage.tsx
function LoginPage() {
  const { register, handleSubmit, formState: { errors, isSubmitting } } = useLoginForm();
  const { mutate: login } = useLoginMutation();

  return (
    <form onSubmit={handleSubmit((data) => login(data))}>
      <Input {...register('email')} error={errors.email?.message} />
      <Input type="password" {...register('password')} error={errors.password?.message} />
      <Button type="submit" loading={isSubmitting}>로그인</Button>
    </form>
  );
}
```

→ schema 분리 + custom hook + 페이지 사용 의 3 단.

## 14. 함정

1. **`<form>` 의 onSubmit 안 e.preventDefault()** — handleSubmit 이 자동. 직접 `onClick={submit}` 하면 안 됨.
2. **`register` 의 typo** — TS 가 못 잡는 경우 있음. zod schema + `z.infer` 로 type-safe.
3. **MUI / antd 에 register 직접** — controlled lib 면 Controller.
4. **default values 의 async** — 처음에 undefined 면 input 이 uncontrolled → controlled 전환 경고. `reset()` 사용.
5. **isValid 의 초기 값** — `mode` 가 'onSubmit' (기본) 이면 첫 submit 전까지 false. 'onChange' 또는 첫 trigger.
6. **watch 남발** — 매 변화 re-render. 진짜 필요한 곳만.
7. **submit 후 reset 안 함** — 같은 폼 다시 쓰려면 `reset()` 호출.

## 15. 외부 자료

- [react-hook-form 공식](https://react-hook-form.com/)
- [zod 공식](https://zod.dev/)
- [@hookform/resolvers](https://github.com/react-hook-form/resolvers)

## 16. 관련

- [[forms]]
- [[../core-concepts/forms-and-controlled-inputs]]
- [[../auth/login-jwt]]
- [[../ui-libraries/antd]]
