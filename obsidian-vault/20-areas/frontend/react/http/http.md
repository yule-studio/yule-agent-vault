---
title: "HTTP in React — fetch / axios / interceptor"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, http, axios, fetch, hub]
---

# HTTP in React

**[[../react|↑ React Hub]]**

> 서버와 통신하는 방법. **fetch (브라우저 기본) vs axios (라이브러리)**.

## 1. 선택 — fetch vs axios

| | fetch | axios |
| --- | --- | --- |
| install | 없음 (브라우저 기본) | yarn add axios |
| 4xx/5xx | resolve (`.ok` 체크) | reject (catch) |
| JSON | `r.json()` 수동 | 자동 |
| interceptor | 없음 | 있음 |
| 취소 | AbortController | AbortController, CancelToken |
| 진행률 | 어려움 | onUploadProgress |
| bundle | 0 | ~13KB |

→ `masterway-dev/*-fe` 가 **axios** 사용.

## 2. fetch 기본

```ts
const r = await fetch('/api/users', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ name: '유철' }),
});

if (!r.ok) throw new Error(`HTTP ${r.status}`);
const data = await r.json();
```

→ fetch 는 **4xx/5xx 도 resolve**. `.ok` 확인 필수.

## 3. axios 기본

```ts
import axios from 'axios';

const r = await axios.post('/api/users', { name: '유철' });
// 자동 JSON, 자동 throw on 4xx/5xx
console.log(r.data);
```

→ 자세히 [[axios-setup]].

## 4. base URL / common header — instance

```ts
// api.ts
import axios from 'axios';

export const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
  headers: { 'Content-Type': 'application/json' },
  timeout: 10000,
  withCredentials: true,    // cookie 포함
});

// 사용
api.get('/users');
api.post('/users', { name: '유철' });
```

## 5. interceptor — 모든 요청 / 응답에 공통 로직

```ts
// 요청 전 token 자동 첨부
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// 응답 401 시 자동 logout
api.interceptors.response.use(
  (r) => r,
  (error) => {
    if (error.response?.status === 401) {
      logout();
    }
    return Promise.reject(error);
  }
);
```

→ 자세히 [[interceptors]].

## 6. react-query 와의 결합

```tsx
const useUsers = () =>
  useQuery({
    queryKey: ['users'],
    queryFn: () => api.get('/users').then(r => r.data),
  });
```

→ HTTP 호출은 axios, 캐싱은 react-query.

## 7. 학습 우선순위

1. **[[axios-setup]]** — instance, baseURL, header, timeout.
2. **[[interceptors]]** — token 첨부, refresh, 에러 처리.
3. **[[../server-state/react-query]]** — 캐싱 / 동기화.
4. **AbortController** — 취소.
5. **FormData / 파일 업로드**.

## 8. 환경별 baseURL

```env
# .env.master
VITE_API_URL=https://api.example.com

# .env.stage
VITE_API_URL=https://api-stage.example.com

# .env.test
VITE_API_URL=https://api-test.example.com
```

```ts
const api = axios.create({ baseURL: import.meta.env.VITE_API_URL });
```

→ `masterway-dev` 의 표준 패턴.

## 9. CORS — 브라우저의 보안 정책

```
브라우저: localhost:5173 → api.example.com 요청
서버: Access-Control-Allow-Origin 헤더 응답
브라우저: 헤더 보고 허용 여부 판단
```

### 흔한 CORS 에러
```
Access to fetch at 'https://api.example.com/users' from origin 'http://localhost:5173'
has been blocked by CORS policy
```

### 해결
- 서버에서 `Access-Control-Allow-Origin` 추가.
- dev 에선 vite proxy 사용:
```ts
// vite.config.ts
server: {
  proxy: {
    '/api': {
      target: 'https://api.example.com',
      changeOrigin: true,
    },
  },
},
```

→ `/api/...` 요청을 vite 가 프록시 → CORS 회피.

## 10. credentials / cookie

```ts
axios.create({ withCredentials: true });   // cookie 자동 포함

fetch('/api', { credentials: 'include' });  // 같음
```

→ httpOnly cookie 기반 인증 (안전) 사용 시 필수.

## 11. FormData — 파일 업로드

```tsx
const fd = new FormData();
fd.append('file', file);
fd.append('description', 'photo');

await api.post('/upload', fd, {
  headers: { 'Content-Type': 'multipart/form-data' },   // axios 가 자동 설정도 함
  onUploadProgress: (e) => {
    setProgress(Math.round((e.loaded / e.total!) * 100));
  },
});
```

## 12. 함정

1. **CORS** — dev 에선 proxy, prod 에선 서버 설정.
2. **token 의 localStorage vs httpOnly cookie** — XSS 시 localStorage 노출. cookie + httpOnly + Secure + SameSite.
3. **error 형식의 통일** — 서버 마다 다르면 interceptor 에서 normalize.
4. **백엔드 404 vs 200 { error }** — 양쪽 다 처리.
5. **요청 race** — 검색어 입력마다 fetch → 응답 순서 보장 X. AbortController 또는 react-query.
6. **timeout 짧음** — file upload 가 잘림. 큰 요청은 별도 instance.

## 13. 외부 자료

- [axios 공식](https://axios-http.com/)
- [MDN Fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)

## 14. 관련

- [[axios-setup]]
- [[interceptors]]
- [[../server-state/react-query]]
- [[../auth/auth]]
