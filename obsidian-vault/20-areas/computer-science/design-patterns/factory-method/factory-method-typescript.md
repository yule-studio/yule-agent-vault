---
title: "Factory Method — TypeScript / Node / NestJS / React"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, factory-method, typescript, node, nestjs, react]
---

# Factory Method — TypeScript / Node / NestJS / React

**[[factory-method|↑ Factory Method hub]]**

## 1. 기본 — 인터페이스 + factory function

```ts
interface Doc {
  render(): void;
}

class PdfDoc implements Doc {
  render() { console.log('pdf'); }
}

class WordDoc implements Doc {
  render() { console.log('word'); }
}

type DocKind = 'pdf' | 'word';

function createDoc(kind: DocKind): Doc {
  switch (kind) {
    case 'pdf':  return new PdfDoc();
    case 'word': return new WordDoc();
  }
}
```

## 2. Static method factory

```ts
class User {
  private constructor(public name: string) {}

  static fromEmail(email: string): User {
    return new User(email.split('@')[0]);
  }

  static guest(): User { return new User('guest'); }
}
```

## 3. Registry pattern

```ts
type ParserClass = { new (): Parser };

abstract class Parser {
  private static registry = new Map<string, ParserClass>();

  static register(ext: string, cls: ParserClass) {
    Parser.registry.set(ext, cls);
  }

  static create(ext: string): Parser {
    const Cls = Parser.registry.get(ext);
    if (!Cls) throw new Error(`unknown: ${ext}`);
    return new Cls();
  }

  abstract parse(data: string): unknown;
}

class JsonParser extends Parser {
  parse(data: string) { return JSON.parse(data); }
}
Parser.register('json', JsonParser);

// 사용
const p = Parser.create('json');
```

## 4. NestJS — `useFactory`

```ts
@Module({
  providers: [
    {
      provide: 'CONFIG',
      useFactory: () => ({                 // factory function
        port: parseInt(process.env.PORT ?? '3000'),
        db: process.env.DATABASE_URL,
      }),
    },
    {
      provide: HttpClient,
      useFactory: (config) => new HttpClient(config.apiUrl),
      inject: ['CONFIG'],                   // dependency injection
    },
  ],
})
export class AppModule {}
```

- NestJS provider 의 `useFactory` 가 표준 factory method.
- 다른 provider 를 `inject` 해서 dependency 와 함께 생성.

## 5. NestJS — Conditional provider

```ts
@Module({
  providers: [
    {
      provide: PaymentService,
      useFactory: () => {
        return process.env.NODE_ENV === 'production'
          ? new StripePaymentService()
          : new FakePaymentService();
      },
    },
  ],
})
```

## 6. React — Component factory (HOC)

```tsx
function withLoading<P>(Component: React.ComponentType<P>) {
  return function WithLoading(props: P & { loading: boolean }) {
    if (props.loading) return <Spinner />;
    return <Component {...props} />;
  };
}

const UserListWithLoading = withLoading(UserList);
```

- Higher-Order Component (HOC) = component 의 factory.

## 7. React — `React.createElement`

```tsx
// JSX
<div>hello</div>

// 컴파일 후
React.createElement('div', null, 'hello')
```

- JSX 가 컴파일러로 factory call 로 변환.
- `createElement` 가 React 의 core factory.

## 8. Node — `http.createServer`

```ts
import http from 'http';

const server = http.createServer((req, res) => {
  res.end('hello');
});
server.listen(3000);
```

- `createServer` 가 server 인스턴스 factory.
- Node std 가 `createX` 패턴 광범위 (`createReadStream`, `createInterface`, etc.).

## 9. Discriminated union + factory function

```ts
type Animal =
  | { kind: 'dog'; name: string; breed: string }
  | { kind: 'cat'; name: string; livesLeft: number };

function createAnimal(kind: 'dog', name: string, breed: string): Animal;
function createAnimal(kind: 'cat', name: string, lives: number): Animal;
function createAnimal(kind: any, ...args: any[]): Animal {
  if (kind === 'dog') return { kind, name: args[0], breed: args[1] };
  return { kind: 'cat', name: args[0], livesLeft: args[1] };
}
```

- TypeScript overload 으로 type-safe factory.

## 10. 함정

1. **switch / if-else 폭주** — 신규 type 추가마다 factory 수정. registry 권장.
2. **NestJS useFactory 의 async** — async factory 는 `useFactory: async () => ...` 로. provider resolution 이 await.
3. **HOC 의 displayName 손실** — debugging 시 `WithLoading(UserList)` 같은 displayName 명시.
4. **JSX createElement 직접 호출** — JSX 가 거의 항상 더 가독성 좋음.
5. **Discriminated union 의 default case** — exhaustiveness check (`never` type) 으로 신규 타입 누락 방지.

## 11. 관련

- [[factory-method]]
- [[factory-method-java]]
- [[factory-method-python]]
- [[factory-method-go]]
