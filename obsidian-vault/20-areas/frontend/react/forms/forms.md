---
title: "Forms — 폼 처리 길잡이"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, forms, react-hook-form, zod, hub]
---

# Forms

**[[../react|↑ React Hub]]**

> 입력 / 검증 / 제출의 표준 = **react-hook-form + zod**.

## 1. 폼 처리의 옵션

| | 특징 | 적합 |
| --- | --- | --- |
| **순수 useState + 수동 validation** | 단순. 모든 게 명시. | 1 ~ 2 필드 |
| **react-hook-form** | uncontrolled 기반 빠름. 매우 강력. | 3+ 필드 / 대규모 |
| **Formik** | 옛 표준. controlled. 느림. | 신규 비추 |
| **antd Form** | antd 와 통합. | antd 쓰면 |
| **MUI 자체 / Chakra** | 라이브러리 내장 | 라이브러리 쓰면 |

→ **react-hook-form 이 사실상 표준**. [[react-hook-form]] 자세히.

## 2. validation 라이브러리

| | 특징 |
| --- | --- |
| **zod** | TS 친화, infer 가능. 가장 인기. |
| **yup** | 역사 깊음. Formik / hook-form 통합. |
| **joi** | 백엔드 출신. FE 도 가능. |
| **valibot** | zod 보다 작음. 신생. |

→ **zod 권장**. type 자동 추론.

## 3. 최소 예 — react-hook-form + zod

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
  const { register, handleSubmit, formState: { errors } } = useForm<FormData>({
    resolver: zodResolver(schema),
  });

  return (
    <form onSubmit={handleSubmit((data) => api.login(data))}>
      <input {...register('email')} />
      {errors.email && <p>{errors.email.message}</p>}

      <input type="password" {...register('password')} />
      {errors.password && <p>{errors.password.message}</p>}

      <button type="submit">로그인</button>
    </form>
  );
}
```

→ 자세히 [[react-hook-form]].

## 4. 일반적인 폼 패턴

### (1) submit 시 API 호출
```tsx
const { mutate: signup, isPending } = useMutation({ mutationFn: api.signup });

<form onSubmit={handleSubmit((data) => signup(data))}>
  ...
  <button type="submit" disabled={isPending}>{isPending ? '...' : '가입'}</button>
</form>
```

### (2) 서버 에러 표시
```tsx
const { setError } = useForm<...>();

try {
  await api.signup(data);
} catch (e) {
  if (e.response?.data.field === 'email') {
    setError('email', { message: e.response.data.message });
  }
}
```

### (3) 폼 초기화 (수정 페이지)
```tsx
useEffect(() => {
  if (user) {
    reset({ name: user.name, email: user.email });   // 기존 값 채움
  }
}, [user]);
```

### (4) controlled (antd / MUI 와 결합)
```tsx
<Controller
  control={control}
  name="email"
  render={({ field }) => <TextField {...field} />}
/>
```

## 5. validation 패턴 예시

```ts
const schema = z.object({
  email: z.string().email(),
  password: z.string().min(8).regex(/[A-Z]/, '대문자 1개 이상'),
  confirmPassword: z.string(),
  age: z.coerce.number().min(18, '18세 이상'),
  agreedTerms: z.literal(true, { errorMap: () => ({ message: '약관 동의 필수' }) }),
}).refine(d => d.password === d.confirmPassword, {
  message: '비밀번호 불일치',
  path: ['confirmPassword'],
});
```

→ z.coerce.number = string → number 자동 변환 (input 은 항상 string).

## 6. masterway-dev 매핑

```bash
grep -E "react-hook-form|zod|yup|formik" ~/masterway-dev/answer-fe/package.json
grep -E "react-hook-form|zod|yup|formik" ~/masterway-dev/job-answer-fe/package.json
```

`answer-fe`: react-hook-form + zod.
`job-answer-fe`: react-hook-form + (zod 또는 antd Form 혼용).

## 7. 학습 우선순위

1. **[[react-hook-form]]** — register / handleSubmit / errors / Controller.
2. **zod schema 작성**.
3. **서버 에러 + setError**.
4. **수정 폼 (default values + reset)**.
5. **multi-step / dynamic field**.

## 8. 함정

1. **controlled vs uncontrolled 혼용** — `value` 와 `defaultValue` 동시 사용 ❌.
2. **register 의 name vs file path** — `name="user.email"` 의 dot path 사용.
3. **antd / MUI 의 onChange API 불일치** — Controller 로 wrap.
4. **submit 시 새로고침** — `<form>` 의 `onSubmit` 안 `e.preventDefault()`. handleSubmit 이 자동.
5. **첫 render 의 default values** — async (api fetch) 값은 reset 으로.
6. **에러 보였다 사라짐** — `mode: 'onChange'` 또는 onBlur 로 즉시 / blur 시.

## 9. 외부 자료

- [react-hook-form 공식](https://react-hook-form.com/)
- [zod 공식](https://zod.dev/)
- [@hookform/resolvers](https://github.com/react-hook-form/resolvers)

## 10. 관련

- [[react-hook-form]]
- [[../core-concepts/forms-and-controlled-inputs]]
- [[../auth/login-jwt]]
