---
title: "State in RN — Frontend 패턴 + AsyncStorage"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, state, async-storage, hub]
---

# State in RN

**[[../react-native|↑ RN Hub]]**

> RN 의 state 관리는 **frontend 와 거의 동일**. 차이 = localStorage 대신 AsyncStorage.

## 1. 상태의 종류 — frontend 와 동일

| | 위치 |
| --- | --- |
| **로컬 UI** | useState |
| **공유 UI** | lift up / context / zustand / jotai |
| **서버 상태** | react-query / SWR |
| **인증 / 세션** | zustand + AsyncStorage |
| **영속 (재실행 후 유지)** | AsyncStorage |

→ 자세히 [[../../../frontend/react/state-management/state-strategy|state-strategy]].

## 2. RN 에서도 같은 라이브러리 사용 가능

- **zustand** — RN 친화. persist 미들웨어가 AsyncStorage 지원.
- **jotai** — 동작.
- **recoil** — 동작. (deprecated)
- **redux toolkit** — 동작.
- **react-query** — RN 에서 가장 흔함.

→ job-answer-app-rn 의 package.json 에는 별도 state lib 없음 (useState + Context + react-navigation 으로 충분).

## 3. AsyncStorage — RN 의 localStorage

```bash
yarn add @react-native-async-storage/async-storage
cd ios && pod install
```

```ts
import AsyncStorage from '@react-native-async-storage/async-storage';

// 저장 — 비동기 + string 만
await AsyncStorage.setItem('token', 'abc123');

// 읽기
const token = await AsyncStorage.getItem('token');   // string | null

// 삭제
await AsyncStorage.removeItem('token');

// 전부 삭제
await AsyncStorage.clear();

// 객체는 JSON 직렬화
await AsyncStorage.setItem('user', JSON.stringify(user));
const user = JSON.parse((await AsyncStorage.getItem('user')) ?? 'null');
```

→ web localStorage 와 차이:
- **비동기** (`await` 필요).
- 메인 thread block X.
- string 만 (객체는 JSON.stringify).

자세히 [[async-storage]].

## 4. zustand + AsyncStorage persist

```ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';

interface AuthStore {
  user: User | null;
  token: string | null;
  setUser: (u: User) => void;
  logout: () => void;
}

export const useAuth = create<AuthStore>()(
  persist(
    (set) => ({
      user: null,
      token: null,
      setUser: (user) => set({ user }),
      logout: () => set({ user: null, token: null }),
    }),
    {
      name: 'auth',
      storage: createJSONStorage(() => AsyncStorage),   // AsyncStorage 사용
    }
  )
);
```

→ 새 앱 실행 후에도 user / token 유지.

## 5. react-query — 서버 상태

```tsx
// frontend 와 거의 동일
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60_000,
      retry: 1,
      refetchOnReconnect: true,           // 네트워크 복귀 시 refetch (RN 특화)
    },
  },
});

<QueryClientProvider client={queryClient}>
  <App />
</QueryClientProvider>
```

### RN 특화 — 앱 foreground/background 감지
```ts
import { focusManager } from '@tanstack/react-query';
import { AppState } from 'react-native';

function onAppStateChange(status: AppStateStatus) {
  focusManager.setFocused(status === 'active');
}

AppState.addEventListener('change', onAppStateChange);
```

→ 앱 백그라운드 → foreground 복귀 시 stale query refetch.

### 네트워크 상태 감지
```bash
yarn add @react-native-community/netinfo
```

```ts
import { onlineManager } from '@tanstack/react-query';
import NetInfo from '@react-native-community/netinfo';

onlineManager.setEventListener((setOnline) => {
  return NetInfo.addEventListener((state) => {
    setOnline(!!state.isConnected);
  });
});
```

자세히 [[../../../frontend/react/server-state/react-query|react-query]] (대부분 그대로 사용).

## 6. Context — 자주 안 변경되는 값

```tsx
// 테마 / 로케일 / 인증 사용자 (변경 거의 X)
const ThemeContext = createContext<'light' | 'dark'>('light');

<ThemeContext.Provider value={theme}>
  <App />
</ThemeContext.Provider>

// 사용
const theme = useContext(ThemeContext);
```

→ 자주 변경되면 zustand 권장.

## 7. job-answer-app-rn 의 패턴 (추정)

```tsx
// 가벼운 앱이면 useState + Context 만으로 충분
// AsyncStorage 로 token 저장 + 인증 상태 관리

// hooks/useAuth.ts
export const useAuth = () => {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    // 앱 시작 시 AsyncStorage 에서 token 읽기
    (async () => {
      const token = await AsyncStorage.getItem('token');
      if (token) {
        try {
          const me = await api.get('/users/me');
          setUser(me.data);
        } catch {
          await AsyncStorage.removeItem('token');
        }
      }
      setIsLoading(false);
    })();
  }, []);

  return { user, isLoading, setUser };
};
```

→ react-query + zustand 도입 시 코드 줄어듬.

## 8. 학습 우선순위

1. **[[async-storage]]** — token / settings 영속화.
2. **react-query** — 서버 상태. [[../../../frontend/react/server-state/react-query|frontend 의 react-query]] 그대로.
3. **zustand + persist** — 전역 상태 + AsyncStorage 통합.
4. **AppState / NetInfo** — RN 특화.

## 9. 함정

1. **localStorage 사용** — RN 에 없음. AsyncStorage.
2. **AsyncStorage 의 동기 호출** — 항상 await.
3. **객체 직접 setItem** — string 만. JSON.stringify.
4. **AsyncStorage 의 limit** — Android 는 6MB 정도. 큰 데이터는 다른 저장소 (MMKV).
5. **token 의 plain text 저장** — 보안 필요 시 keychain (`react-native-keychain`).
6. **multi-app token leak** — Android 의 같은 keystore 사용 시 주의.

## 10. 더 빠른 저장소 — MMKV

```bash
yarn add react-native-mmkv
```

```ts
import { MMKV } from 'react-native-mmkv';

export const storage = new MMKV();

storage.set('token', 'abc');               // 동기!
const token = storage.getString('token');   // 동기
storage.delete('token');
```

→ AsyncStorage 보다 30x 빠름. 동기. RN 신규 추천.

## 11. 보안 저장소 — react-native-keychain

```bash
yarn add react-native-keychain
```

```ts
import * as Keychain from 'react-native-keychain';

await Keychain.setGenericPassword('user', 'token');
const creds = await Keychain.getGenericPassword();
await Keychain.resetGenericPassword();
```

→ token / 패스워드처럼 민감한 데이터. iOS Keychain / Android Keystore.

## 12. 다음 단계

- [[async-storage]] — AsyncStorage 깊이
- [[../networking/networking]] — 통신

## 13. 관련

- [[../react-native]]
- [[../../../frontend/react/state-management/state-management|frontend state]]
- [[../../../frontend/react/server-state/react-query|react-query]]
