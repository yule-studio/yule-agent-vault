---
title: "axios — instance / baseURL / header / 에러"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, axios, http, instance]
---

# axios setup

**[[http|↑ HTTP Hub]]**

> `masterway-dev/*-fe` 가 사용. instance 1 개 + interceptor 가 표준.

## 1. 설치

```bash
yarn add axios
```

## 2. 기본 사용

```ts
import axios from 'axios';

// GET
const { data } = await axios.get('/api/users');

// POST
const { data } = await axios.post('/api/users', { name: '유철' });

// PUT / PATCH / DELETE
await axios.put('/api/users/1', { name: '유미' });
await axios.patch('/api/users/1', { name: '유미' });
await axios.delete('/api/users/1');
```

→ JSON 직렬화 자동.

## 3. instance — 표준 패턴

```ts
// lib/api.ts
import axios from 'axios';

export const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
  timeout: 10_000,
  headers: { 'Content-Type': 'application/json' },
  withCredentials: true,
});
```

→ 매 호출 baseURL 안 적어도 됨. 환경 변수로 분기.

### 외부 / 내부 API 분리
```ts
export const apiInternal = axios.create({ baseURL: '/api' });
export const apiKakao = axios.create({ baseURL: 'https://kapi.kakao.com' });
```

## 4. 환경 변수 — Vite 의 `import.meta.env`

```env
# .env (또는 .env.master / .env.stage / .env.test)
VITE_API_URL=https://api.example.com
```

```ts
const api = axios.create({ baseURL: import.meta.env.VITE_API_URL });
```

→ `VITE_` prefix 만 클라이언트로 노출.

## 5. axios 응답 구조

```ts
const response = await axios.get('/api/users');

response.data         // 실제 body (가장 많이 쓰는 것)
response.status       // 200
response.statusText   // "OK"
response.headers      // 응답 헤더
response.config       // 요청 config
```

→ 거의 `r.data` 만 씀. helper 패턴:
```ts
const getJson = <T>(url: string) => api.get<T>(url).then(r => r.data);
const users = await getJson<User[]>('/users');
```

## 6. TypeScript 타입

```ts
interface User { id: number; name: string; }

// 응답 타입 명시
const r = await api.get<User[]>('/users');
// r.data: User[]

// POST 의 body / response 동시 명시
const r = await api.post<User, AxiosResponse<User>, { name: string }>(
  '/users',
  { name: '유철' }
);
```

→ 보통 `api.get<User[]>('/users').then(r => r.data)` 패턴.

## 7. 에러 처리

```ts
import axios from 'axios';

try {
  await api.get('/users');
} catch (error) {
  if (axios.isAxiosError(error)) {
    console.log(error.response?.status);   // 4xx / 5xx
    console.log(error.response?.data);     // 서버 에러 body
    console.log(error.message);
  } else {
    console.log('네트워크 또는 알 수 없는 에러');
  }
}
```

### 에러의 3 가지 경우
1. **서버 응답 4xx/5xx** — `error.response` 있음.
2. **요청은 갔지만 응답 없음 (timeout / 네트워크)** — `error.request` 있음, `error.response` 없음.
3. **요청 만들기 실패 (config 오류)** — `error.message`.

## 8. timeout

```ts
const api = axios.create({ timeout: 10_000 });

// 특정 요청만 다른 timeout
api.get('/big-file', { timeout: 60_000 });
```

## 9. 취소 — AbortController

```ts
const controller = new AbortController();

const fetchData = async () => {
  try {
    await api.get('/users', { signal: controller.signal });
  } catch (e) {
    if (axios.isCancel(e)) console.log('취소됨');
  }
};

// 취소
controller.abort();
```

```tsx
// useEffect 안
useEffect(() => {
  const controller = new AbortController();
  api.get('/data', { signal: controller.signal }).then(...);
  return () => controller.abort();
}, []);
```

→ unmount 또는 새 fetch 시 이전 cancel. react-query 사용 시 자동.

## 10. params (query string)

```ts
await api.get('/users', { params: { page: 2, sort: 'name' } });
// → GET /users?page=2&sort=name
```

→ 직접 `?...` 안 만들어도 됨.

## 11. 다른 Content-Type — form-urlencoded / multipart

```ts
// form-urlencoded
import qs from 'qs';
await api.post('/login', qs.stringify({ email, password }), {
  headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
});

// multipart (FormData)
const fd = new FormData();
fd.append('file', file);
await api.post('/upload', fd);   // Content-Type 자동
```

## 12. answer-fe / job-answer-fe 패턴

```ts
// lib/api/axios.ts
import axios from 'axios';

export const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
  timeout: 10_000,
  withCredentials: true,
});

// interceptor 설정 → [[interceptors]]
```

```ts
// lib/api/user.ts
import { api } from './axios';

export const userApi = {
  getMe: () => api.get<User>('/users/me').then(r => r.data),
  login: (body: LoginDto) => api.post<{ token: string }>('/auth/login', body).then(r => r.data),
  signup: (body: SignupDto) => api.post('/auth/signup', body).then(r => r.data),
};
```

→ 도메인별 모듈로 정리.

## 13. 함정

1. **`api` 안 만들고 매번 axios.get** — baseURL / token 매번 적어야.
2. **`axios.defaults.headers.common[...]`** — 전역 mutation. instance 권장.
3. **timeout 0** — 무한 대기 = 사용자 경험 나쁨.
4. **에러 type guard 빠뜨림** — `axios.isAxiosError(e)` 로 좁혀야 `.response` 접근.
5. **interceptor 의 token 의 stale closure** — token 변경 후 갱신 안 됨. 항상 storage 에서 read.
6. **`withCredentials` 누락** — cookie 인증 안 됨.

## 14. 외부 자료

- [axios 공식](https://axios-http.com/)
- [Vite env 변수](https://vitejs.dev/guide/env-and-mode.html)

## 15. 관련

- [[http]]
- [[interceptors]]
- [[../auth/auth]]
