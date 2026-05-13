---
title: "Singleton — TypeScript / Node / React / NestJS"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, gof, creational, singleton, typescript, node, react, nestjs]
---

# Singleton — TypeScript / Node / React / NestJS

**[[singleton|↑ Singleton hub]]**

## 1. Module 자체가 Singleton (Node / ES Module)

```ts
// logger.ts
class LoggerImpl {
  log(msg: string) { console.log(msg); }
}

export const logger = new LoggerImpl();
```

```ts
// 다른 파일
import { logger } from './logger';
logger.log('hello');
```

- ES module 은 한 번만 evaluate 됨.
- 모든 import 가 같은 인스턴스 참조.
- **TypeScript / Node 의 가장 흔한 Singleton**.

## 2. Class + static instance

```ts
class Logger {
  private static instance: Logger;
  private constructor() {}

  static getInstance(): Logger {
    if (!Logger.instance) {
      Logger.instance = new Logger();
    }
    return Logger.instance;
  }

  log(msg: string) { console.log(msg); }
}

Logger.getInstance().log('hello');
```

- private constructor 로 외부 new 방지.
- TypeScript 가 컴파일 타임에 검증.

## 3. Lazy + 비동기 초기화

```ts
class Database {
  private static instance: Database | null = null;
  private static initPromise: Promise<Database> | null = null;

  private constructor(private pool: Pool) {}

  static async getInstance(): Promise<Database> {
    if (Database.instance) return Database.instance;
    if (Database.initPromise) return Database.initPromise;

    Database.initPromise = (async () => {
      const pool = await createPool();
      Database.instance = new Database(pool);
      return Database.instance;
    })();
    return Database.initPromise;
  }
}
```

- 비동기 초기화 중복 방지.
- Promise 도 cache.

## 4. NestJS — Provider 의 기본 Singleton

```ts
// logger.service.ts
import { Injectable } from '@nestjs/common';

@Injectable()
export class LoggerService {
  log(msg: string) { console.log(msg); }
}

// app.module.ts
@Module({
  providers: [LoggerService],
  exports: [LoggerService],
})
export class AppModule {}

// order.service.ts
@Injectable()
export class OrderService {
  constructor(private logger: LoggerService) {}   // DI

  placeOrder(...) {
    this.logger.log('order placed');
  }
}
```

- NestJS 의 `@Injectable` provider 기본 scope = singleton.
- 모듈이 같은 인스턴스를 다른 service 에 주입.

### Scope 명시

```ts
@Injectable({ scope: Scope.DEFAULT })    // singleton (default)
@Injectable({ scope: Scope.REQUEST })    // HTTP 요청 별 새 인스턴스
@Injectable({ scope: Scope.TRANSIENT })  // inject 마다 새 인스턴스
```

## 5. NestJS — 1 process 에 1 instance 보장의 함정

- NestJS 의 singleton 은 **process 단위**.
- multi-instance (PM2 cluster / Docker replica) 환경에서는 instance N 개 = N 인스턴스.
- 진정한 글로벌 = Redis / DB / external service.

## 6. React — Context 가 Singleton 역할

```tsx
// AuthContext.tsx
import { createContext, useContext, useState } from 'react';

interface AuthState {
  user: User | null;
  login: (u: User) => void;
}

const AuthContext = createContext<AuthState | null>(null);

export function AuthProvider({ children }) {
  const [user, setUser] = useState<User | null>(null);
  return (
    <AuthContext.Provider value={{ user, login: setUser }}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error('AuthProvider missing');
  return ctx;
}
```

```tsx
// App.tsx — provider 가 1 곳에만
<AuthProvider>
  <Routes>
    ...
  </Routes>
</AuthProvider>
```

- React tree 의 root 에 Provider 1 개 → 모든 자식이 같은 state 참조.
- "Singleton-like state" 의 React 답.

## 7. React — Zustand / Redux store (전역 Singleton state)

```ts
// store.ts
import { create } from 'zustand';

interface AppState {
  count: number;
  inc: () => void;
}

export const useAppStore = create<AppState>((set) => ({
  count: 0,
  inc: () => set((s) => ({ count: s.count + 1 })),
}));
```

```tsx
// Component
const count = useAppStore((s) => s.count);
const inc = useAppStore((s) => s.inc);
```

- module 의 export 가 곧 store singleton.
- 어디서든 같은 state.
- Redux store, Zustand, Jotai 모두 같은 패턴.

## 8. Node — DB / Cache pool 의 Singleton

```ts
// db.ts
import { Pool } from 'pg';

export const pgPool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,
});

export async function query(text: string, params?: any[]) {
  return pgPool.query(text, params);
}
```

- pool 자체가 connection 풀.
- module 1 회 evaluate → 1 pool.
- worker process 별 pool 1 개.

## 9. NestJS — Configuration Singleton

```ts
// config.module.ts
@Module({
  providers: [
    {
      provide: 'CONFIG',
      useFactory: () => loadConfig(),    // 1 회만 호출
    },
  ],
  exports: ['CONFIG'],
})
export class ConfigModule {}

// 사용
@Injectable()
export class AppService {
  constructor(@Inject('CONFIG') private config: Config) {}
}
```

## 10. 함정 (TypeScript / Node specific)

1. **Module hot-reload (webpack / vite)** — dev server 가 module 다시 evaluate 시 Singleton state 잃음.
2. **Multi-bundle (server + client)** — Next.js 의 server 와 client 가 각자 module 가짐 → 다른 인스턴스.
3. **Worker thread / cluster** — Node cluster 의 각 worker = process. Singleton 이 worker 수만큼.
4. **React Context provider 미포함** — `useAuth()` 호출 시 `useContext` 가 null. error guard 필수.
5. **Class 의 prototype pollution** — Singleton 의 method 수정이 다른 모듈에 전파.
6. **NestJS request-scoped 의 inject 문제** — request scope provider 를 singleton scope provider 에 inject 하려면 `ContextIdFactory` 등 별도 처리.
7. **Type narrowing** — `if (!Singleton.instance)` 가 multi-call 시 race. async 작업이면 lock 필요.

## 11. 권장 — 현대 TypeScript 답

1. **node 의 service / pool / config** = ES module export.
2. **NestJS app** = `@Injectable()` (default singleton).
3. **React global state** = Context + Zustand/Redux/Jotai store.
4. **테스트 친화** = constructor injection 또는 module mocking (`jest.mock`).

→ 옛 `class.getInstance()` 패턴은 거의 안 씀. ES module 또는 DI 가 더 idiomatic.

## 12. 관련

- [[singleton]]
- [[singleton-java]]
- [[singleton-python]]
- [[singleton-go]]
