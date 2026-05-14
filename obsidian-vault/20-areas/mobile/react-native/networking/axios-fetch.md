---
title: "axios / fetch in RN — 거의 frontend 와 동일"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, axios, fetch, http]
---

# axios / fetch in RN

**[[networking|↑ Networking Hub]]**

> [[../../../frontend/react/http/axios-setup|frontend 의 axios-setup]] 와 99% 동일. RN 특화 차이만 다룸.

## 1. 설치

```bash
yarn add axios
```

→ pod install 불필요 (pure JS lib).

## 2. instance

```ts
// lib/api.ts
import axios from 'axios';

export const api = axios.create({
  baseURL: process.env.API_URL ?? 'https://api.example.com',
  timeout: 10_000,
  withCredentials: true,
});
```

### 환경 변수 — RN
```bash
yarn add -D react-native-dotenv
```

```js
// babel.config.js
plugins: [
  ['module:react-native-dotenv', { moduleName: '@env', path: '.env' }],
]
```

```env
# .env
API_URL=https://api.example.com
```

```ts
import { API_URL } from '@env';
export const api = axios.create({ baseURL: API_URL });
```

## 3. interceptor — token + 401 refresh

→ frontend 와 동일. **차이는 token storage 가 AsyncStorage (비동기)**.

```ts
import AsyncStorage from '@react-native-async-storage/async-storage';

api.interceptors.request.use(async (config) => {     // async!
  const token = await AsyncStorage.getItem('access_token');
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

let isRefreshing = false;
let waitQueue: Array<(t: string) => void> = [];

api.interceptors.response.use(
  (r) => r,
  async (error) => {
    const original = error.config;
    if (error.response?.status !== 401 || original._retry) {
      return Promise.reject(error);
    }

    if (isRefreshing) {
      return new Promise((resolve) => {
        waitQueue.push((token) => {
          original.headers.Authorization = `Bearer ${token}`;
          resolve(api(original));
        });
      });
    }

    original._retry = true;
    isRefreshing = true;

    try {
      const refresh = await AsyncStorage.getItem('refresh_token');
      const { data } = await axios.post('/auth/refresh', { refreshToken: refresh });
      await AsyncStorage.setItem('access_token', data.accessToken);

      waitQueue.forEach(cb => cb(data.accessToken));
      waitQueue = [];

      original.headers.Authorization = `Bearer ${data.accessToken}`;
      return api(original);
    } catch (e) {
      await AsyncStorage.multiRemove(['access_token', 'refresh_token']);
      // RN 에선 window.location 안 됨
      // 대신 navigation reset 또는 global event
      // 자세히 [[../auth/auth]]
      return Promise.reject(e);
    } finally {
      isRefreshing = false;
    }
  }
);
```

→ **`async` 콜백** 가능 (axios 가 지원).

## 4. axios 응답 처리

```ts
try {
  const { data } = await api.get<User>('/users/me');
  setUser(data);
} catch (e) {
  if (axios.isAxiosError(e)) {
    if (e.response) {
      console.log(e.response.status, e.response.data);
    } else if (e.request) {
      console.log('네트워크 / timeout');
    }
  }
}
```

## 5. AbortController — 취소

```ts
const controller = new AbortController();
api.get('/users', { signal: controller.signal });
controller.abort();
```

→ unmount 시 자동 cancel:
```tsx
useEffect(() => {
  const controller = new AbortController();
  api.get('/data', { signal: controller.signal }).then(...);
  return () => controller.abort();
}, []);
```

→ react-query 사용 시 자동.

## 6. file 업로드

```ts
const fd = new FormData();
fd.append('file', {
  uri: localUri,                              // 'file:///...'
  type: 'image/jpeg',
  name: 'photo.jpg',
} as any);

const { data } = await api.post('/upload', fd, {
  headers: { 'Content-Type': 'multipart/form-data' },
  onUploadProgress: (e) => {
    if (e.total) setProgress(Math.round((e.loaded / e.total) * 100));
  },
});
```

→ web 의 FormData 와 차이: RN 은 **`{ uri, type, name }` 객체**.

## 7. timeout / retry

```ts
const api = axios.create({ timeout: 10_000 });

// retry
import axiosRetry from 'axios-retry';
axiosRetry(api, {
  retries: 3,
  retryDelay: (count) => count * 1000,
  retryCondition: (e) => {
    return axiosRetry.isNetworkOrIdempotentRequestError(e);
  },
});
```

## 8. fetch 사용 시

```ts
const r = await fetch('https://api.example.com/users', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ name: '유철' }),
});
if (!r.ok) throw new Error(`HTTP ${r.status}`);
const data = await r.json();
```

→ axios 가 일반적으로 더 편함. fetch 는 의존성 minimize 필요한 경우.

## 9. react-query 와 결합

```tsx
import { useQuery } from '@tanstack/react-query';

export const useMe = () =>
  useQuery({
    queryKey: ['me'],
    queryFn: () => api.get('/users/me').then(r => r.data),
    enabled: !!token,
  });
```

→ frontend 와 동일.

## 10. domain 별 API 모듈

```ts
// lib/api/user.ts
export const userApi = {
  getMe: () => api.get<User>('/users/me').then(r => r.data),
  update: (body: UpdateUser) => api.patch<User>('/users/me', body).then(r => r.data),
  delete: () => api.delete('/users/me'),
};
```

```ts
// 사용
import { userApi } from '@/lib/api/user';
const user = await userApi.getMe();
```

## 11. job-answer-app-rn 의 패턴

```ts
// lib/api/axios.ts (추정)
import axios from 'axios';
import AsyncStorage from '@react-native-async-storage/async-storage';
import { API_URL } from '@env';

export const api = axios.create({
  baseURL: API_URL,
  timeout: 10_000,
});

api.interceptors.request.use(async (config) => {
  const token = await AsyncStorage.getItem('access_token');
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

api.interceptors.response.use(
  r => r,
  async (error) => {
    if (error.response?.status === 401) {
      // refresh 로직
    }
    return Promise.reject(error);
  }
);
```

## 12. 함정

1. **interceptor 안 sync token read** — AsyncStorage 는 비동기. `await` 필수.
2. **fetch 의 4xx/5xx resolve** — `.ok` 체크. axios 권장.
3. **timeout 짧음** — 모바일 네트워크 약함. 10s+.
4. **FormData 의 file 객체** — RN 은 `{ uri, type, name }`. blob 아님.
5. **dev http on Android 9+** — cleartext 허용 필요.
6. **window.location 사용** — 없음. navigation.reset 또는 event.
7. **base URL trailing /** — `/api/` + `users` = `/api/users` OK. `/api` + `/users` = `/api/users` OK. 일관성.

## 13. 다음 단계

- [[websocket-stomp]] — 실시간
- [[../auth/auth]] — 인증

## 14. 관련

- [[networking]]
- [[../state/async-storage]]
- [[../../../frontend/react/http/axios-setup|frontend axios-setup]]
- [[../../../frontend/react/http/interceptors|frontend interceptors]]
