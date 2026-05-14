---
title: "axios interceptor — token, refresh, 에러 통일"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, axios, interceptors, jwt, refresh-token]
---

# axios interceptor

**[[http|↑ HTTP Hub]]**

> 모든 요청 / 응답에 공통 로직을 끼우는 패턴.

## 1. 두 종류

```ts
// 1. 요청 전 — 보내기 직전 config 수정
api.interceptors.request.use(
  (config) => { /* config 수정 */ return config; },
  (error) => Promise.reject(error)
);

// 2. 응답 후 — 받은 응답을 가공 또는 에러 처리
api.interceptors.response.use(
  (response) => response,
  (error) => { /* 에러 처리 */ return Promise.reject(error); }
);
```

## 2. token 자동 첨부 (기본)

```ts
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('access_token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});
```

→ 모든 요청에 자동 Authorization 헤더.

## 3. 응답 에러 통일

```ts
api.interceptors.response.use(
  (r) => r,
  (error) => {
    if (axios.isAxiosError(error)) {
      // 1. 네트워크 / timeout
      if (!error.response) {
        toast.error('네트워크 연결 확인');
        return Promise.reject(error);
      }

      // 2. 401 → 로그아웃 + login 페이지
      if (error.response.status === 401) {
        logout();
        window.location.href = '/login';
        return Promise.reject(error);
      }

      // 3. 403
      if (error.response.status === 403) {
        toast.error('권한 없음');
      }

      // 4. 5xx
      if (error.response.status >= 500) {
        toast.error('서버 오류. 잠시 후 다시 시도');
      }

      // 5. 일반 4xx — 서버 메시지 표시
      const message = error.response.data?.message ?? '요청 실패';
      toast.error(message);
    }
    return Promise.reject(error);
  }
);
```

## 4. Refresh Token — 401 시 자동 재발급 + 재시도

```ts
let isRefreshing = false;
let waitingQueue: Array<(token: string) => void> = [];

api.interceptors.response.use(
  (r) => r,
  async (error) => {
    const original = error.config;

    if (error.response?.status === 401 && !original._retry) {
      if (isRefreshing) {
        // 동시 401 들 → 대기열
        return new Promise((resolve) => {
          waitingQueue.push((newToken) => {
            original.headers.Authorization = `Bearer ${newToken}`;
            resolve(api(original));
          });
        });
      }

      original._retry = true;
      isRefreshing = true;

      try {
        const refresh = localStorage.getItem('refresh_token');
        const { data } = await axios.post('/auth/refresh', { refresh });
        const newToken = data.accessToken;

        localStorage.setItem('access_token', newToken);
        api.defaults.headers.Authorization = `Bearer ${newToken}`;
        original.headers.Authorization = `Bearer ${newToken}`;

        // 대기열 깨우기
        waitingQueue.forEach(cb => cb(newToken));
        waitingQueue = [];

        return api(original);          // 재요청
      } catch (e) {
        // refresh 실패 → logout
        localStorage.clear();
        window.location.href = '/login';
        return Promise.reject(e);
      } finally {
        isRefreshing = false;
      }
    }

    return Promise.reject(error);
  }
);
```

→ JWT + Refresh 패턴의 정석. 동시 401 처리 (queue) 가 핵심.

## 5. 로그 / 모니터링

```ts
api.interceptors.request.use((config) => {
  console.log(`→ ${config.method?.toUpperCase()} ${config.url}`);
  return config;
});

api.interceptors.response.use(
  (r) => {
    console.log(`← ${r.status} ${r.config.url}`);
    return r;
  },
  (e) => {
    console.error(`✗ ${e.config?.url}: ${e.response?.status ?? 'network'}`);
    return Promise.reject(e);
  }
);
```

→ Sentry 같은 도구 연동:
```ts
api.interceptors.response.use(
  r => r,
  e => {
    if (e.response?.status >= 500) {
      Sentry.captureException(e);
    }
    return Promise.reject(e);
  }
);
```

## 6. 응답 변환 — `r.data` 자동 추출

```ts
// 매번 r.data 쓰지 않고 싶을 때
api.interceptors.response.use((r) => r.data);   // ⚠️ 타입 깨짐

// 또는 helper 함수
export const apiGet = <T>(url: string) => api.get<T>(url).then(r => r.data);
```

→ 자동 추출은 axios 타입 정의와 안 맞아 헷갈림. **helper 함수가 더 깔끔**.

## 7. answer-fe / job-answer-fe 패턴

```ts
// lib/api/axios.ts
import axios from 'axios';
import { tokenStorage } from '@/lib/auth/tokenStorage';

export const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
  withCredentials: true,
});

// 요청 — token
api.interceptors.request.use((config) => {
  const token = tokenStorage.get();
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// 응답 — 401 refresh
api.interceptors.response.use(
  (r) => r,
  async (error) => {
    if (error.response?.status === 401 && !error.config._retry) {
      // refresh 로직
    }
    return Promise.reject(error);
  }
);
```

## 8. 여러 instance — 다른 정책

```ts
// 일반 — token 첨부 + 401 refresh
export const api = axios.create({ baseURL: '...' });
api.interceptors.request.use(addToken);
api.interceptors.response.use(r => r, refreshOn401);

// 인증 안 거치는 — login / signup
export const publicApi = axios.create({ baseURL: '...' });

// 외부 — kakao / 결제
export const apiKakao = axios.create({ baseURL: '...' });
```

## 9. interceptor 의 함정

1. **중복 등록** — module re-import 또는 HMR 로 interceptor 가 N 번 등록. 등록은 module top-level 1 회만.
2. **stale closure** — interceptor 함수 안의 `useUser` hook 값은 옛 값. **항상 store / localStorage 에서 read**.
3. **Refresh 의 무한 loop** — refresh API 도 401 받으면 또 refresh 시도. `_retry` 플래그로 막기.
4. **동시 401 의 N 회 refresh** — 위 queue 패턴 필요. (없으면 token 이 N 번 교체)
5. **`window.location.href = '/login'`** — SPA 의 navigate 가 아니라 페이지 reload. 의도면 OK, 아니면 react-router 의 navigate 사용 (단 interceptor 에선 hook 못 씀 → 별도 store / event 발행).
6. **모든 4xx 를 toast** — 의도된 4xx (예: validation) 도 toast → 사용자 혼란. 케이스별 분기.

## 10. interceptor 외 — react-query 의 `onError`

```tsx
const qc = new QueryClient({
  defaultOptions: {
    queries: {
      onError: (error) => {
        // 모든 query 의 에러 처리
      },
    },
    mutations: {
      onError: (error) => {
        // 모든 mutation 의 에러 처리
      },
    },
  },
});
```

→ axios interceptor + react-query onError 함께 가능. 책임 분리:
- axios: token / 401 refresh / 네트워크.
- react-query: stale 캐시 / 도메인 별 처리.

## 11. 외부 자료

- [axios interceptor 공식](https://axios-http.com/docs/interceptors)
- [Refresh Token 정석 패턴](https://thecode.pub/jwt-refresh-token-axios-interceptor) (참고용)

## 12. 관련

- [[axios-setup]]
- [[../auth/auth]]
- [[../auth/login-jwt]]
- [[../server-state/react-query]]
