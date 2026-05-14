---
title: "AsyncStorage — RN 의 localStorage 대체"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, async-storage, storage]
---

# AsyncStorage

**[[state|↑ State Hub]]**

> RN 의 key-value 영속 저장소. **비동기** + string only.

## 1. 설치

```bash
yarn add @react-native-async-storage/async-storage
cd ios && pod install
```

## 2. 기본 API

```ts
import AsyncStorage from '@react-native-async-storage/async-storage';

// 저장
await AsyncStorage.setItem('key', 'value');

// 읽기
const value = await AsyncStorage.getItem('key');    // string | null

// 삭제
await AsyncStorage.removeItem('key');

// 전부 삭제 (조심)
await AsyncStorage.clear();

// 모든 key
const keys = await AsyncStorage.getAllKeys();
const items = await AsyncStorage.multiGet(keys);
```

## 3. 객체 / 배열 — JSON 직렬화

```ts
// 저장
const user = { id: 1, name: '유철' };
await AsyncStorage.setItem('user', JSON.stringify(user));

// 읽기
const raw = await AsyncStorage.getItem('user');
const user = raw ? JSON.parse(raw) as User : null;
```

→ 매번 stringify / parse. 헬퍼 작성 권장.

## 4. type-safe wrapper

```ts
// lib/storage.ts
class TypedStorage {
  async get<T>(key: string): Promise<T | null> {
    const raw = await AsyncStorage.getItem(key);
    return raw ? (JSON.parse(raw) as T) : null;
  }

  async set<T>(key: string, value: T): Promise<void> {
    await AsyncStorage.setItem(key, JSON.stringify(value));
  }

  async remove(key: string): Promise<void> {
    await AsyncStorage.removeItem(key);
  }
}

export const storage = new TypedStorage();
```

```ts
await storage.set('user', { id: 1, name: '유철' });
const user = await storage.get<User>('user');
```

## 5. 자주 쓰는 키

```ts
const StorageKeys = {
  TOKEN: 'auth_token',
  REFRESH_TOKEN: 'refresh_token',
  USER: 'user',
  LOCALE: 'locale',
  THEME: 'theme',
  ONBOARDING_DONE: 'onboarding_done',
} as const;
```

→ 상수 객체로 typo 방지.

## 6. 인증 token 저장 — 표준 패턴

```ts
// lib/auth/tokenStorage.ts
import AsyncStorage from '@react-native-async-storage/async-storage';

export const tokenStorage = {
  getAccess: () => AsyncStorage.getItem('access_token'),
  setAccess: (t: string) => AsyncStorage.setItem('access_token', t),
  getRefresh: () => AsyncStorage.getItem('refresh_token'),
  setRefresh: (t: string) => AsyncStorage.setItem('refresh_token', t),
  clear: () => AsyncStorage.multiRemove(['access_token', 'refresh_token']),
};
```

```ts
// axios interceptor
api.interceptors.request.use(async (config) => {
  const token = await tokenStorage.getAccess();
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});
```

→ interceptor 의 콜백이 async 임에 주의 (web 의 sync 와 차이).

## 7. 한 번 보여줄 화면 — onboarding 플래그

```ts
useEffect(() => {
  (async () => {
    const done = await AsyncStorage.getItem('onboarding_done');
    if (!done) {
      navigation.navigate('Onboarding');
    }
  })();
}, []);

// 끝나면
await AsyncStorage.setItem('onboarding_done', '1');
```

## 8. setting / preferences

```ts
// 사용자 설정 객체
const settings = {
  language: 'ko',
  theme: 'light',
  notification: true,
};

await storage.set('settings', settings);
```

## 9. zustand + AsyncStorage persist

```ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';

export const useAuth = create(
  persist<AuthState>(
    (set) => ({
      user: null,
      token: null,
      setUser: (u) => set({ user: u }),
      logout: () => set({ user: null, token: null }),
    }),
    {
      name: 'auth',                                    // 키
      storage: createJSONStorage(() => AsyncStorage),
      partialize: (state) => ({                        // 일부만 저장
        token: state.token,
        user: state.user,
      }),
    }
  )
);
```

→ 자동 영속화. 새 실행 시 자동 복원.

## 10. multiGet / multiSet — 일괄

```ts
// 여러 키 한 번에
const items = await AsyncStorage.multiGet(['k1', 'k2', 'k3']);
// [['k1', 'v1'], ['k2', 'v2'], ['k3', null]]

await AsyncStorage.multiSet([
  ['k1', 'v1'],
  ['k2', 'v2'],
]);

await AsyncStorage.multiRemove(['k1', 'k2']);
```

→ 효율 (개별 호출 N 번 보다 빠름).

## 11. error 처리

```ts
try {
  await AsyncStorage.setItem('key', 'value');
} catch (e) {
  // 저장 실패 (disk full / 권한 등)
  console.error(e);
}
```

→ 실제로는 거의 실패 X. critical data 만 catch.

## 12. 용량 제한

| | 제한 |
| --- | --- |
| iOS | (이론상 무제한, 실제 GB 단위) |
| Android | 6MB (기본) — Java exception |

→ 큰 데이터는 [[#13-mmkv|MMKV]] 또는 파일 (`react-native-fs`).

### Android limit 늘리기
`android/gradle.properties`:
```
AsyncStorage_db_size_in_MB=10
```

## 13. MMKV — 더 빠른 대안

```bash
yarn add react-native-mmkv
```

```ts
import { MMKV } from 'react-native-mmkv';

export const storage = new MMKV();

storage.set('token', 'abc');                   // ✅ 동기!
const token = storage.getString('token');       // string | undefined
storage.delete('token');

// 객체
storage.set('user', JSON.stringify(user));
const user = JSON.parse(storage.getString('user') ?? 'null');
```

→ AsyncStorage 보다 **30x 빠름**. **동기**. UI 안 막음.
→ 신규 코드 / 마이그레이션 강력 추천.

## 14. AsyncStorage → MMKV 마이그레이션

```ts
import AsyncStorage from '@react-native-async-storage/async-storage';
import { MMKV } from 'react-native-mmkv';

const mmkv = new MMKV();

async function migrateOnce() {
  if (mmkv.getBoolean('_migrated')) return;
  const keys = await AsyncStorage.getAllKeys();
  const items = await AsyncStorage.multiGet(keys);
  items.forEach(([k, v]) => v && mmkv.set(k, v));
  mmkv.set('_migrated', true);
  await AsyncStorage.clear();
}
```

## 15. 보안 — token / 비밀번호

```bash
yarn add react-native-keychain
```

```ts
import * as Keychain from 'react-native-keychain';

await Keychain.setGenericPassword('user', 'token-value');

const creds = await Keychain.getGenericPassword();
if (creds) {
  const { username, password } = creds;       // 'user', 'token-value'
}

await Keychain.resetGenericPassword();
```

→ iOS Keychain / Android EncryptedSharedPreferences. **민감 데이터는 AsyncStorage X**.

## 16. 함정

1. **AsyncStorage 의 await 잊음** — Promise 자체가 string 인 줄 알고 사용.
2. **null check 누락** — 없으면 null. `JSON.parse(null)` 던짐.
3. **객체 직접 set** — `[object Object]` 저장됨. stringify 필수.
4. **Android 6MB limit** — 큰 데이터 시 silent fail. MMKV / 파일.
5. **token plain text** — XSS 가 없어도 rooted device. Keychain.
6. **logout 시 clear** — `clear()` 가 모든 키 삭제. 일부 보존 필요 시 `multiRemove`.
7. **dev mode 의 reload** — AsyncStorage 는 reload 후 유지. clear 안 됨.

## 17. 디버깅

```bash
# 모든 키 확인
console.log(await AsyncStorage.getAllKeys());

# Flipper 또는 React DevTools 의 AsyncStorage plugin
```

## 18. 다음 단계

- [[../auth/auth]] — token 저장과 함께
- [[../networking/networking]] — interceptor 의 token

## 19. 외부 자료

- [@react-native-async-storage/async-storage](https://react-native-async-storage.github.io/async-storage/)
- [react-native-mmkv](https://github.com/mrousavy/react-native-mmkv)
- [react-native-keychain](https://github.com/oblador/react-native-keychain)

## 20. 관련

- [[state]]
- [[../auth/auth]]
- [[../networking/networking]]
