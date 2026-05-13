---
title: "Decorator"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, gof, structural, decorator, wrapper, middleware]
---

# Decorator

**[[../design-patterns|↑ 디자인 패턴]]** · GoF Structural

## 1. 한 줄

**객체에 동적으로 책임 추가.** 상속 없이 layer 를 쌓아 행동 확장.

## 2. 언제 쓰는가

- **logging / timing / retry / cache / auth 같은 cross-cutting concern**.
- **stream 처리** — BufferedReader 가 FileReader 를 wrap.
- **middleware chain** — HTTP pipeline.
- **subclass 폭발 회피** — Coffee × (milk × sugar × …) = 2^N subclass 대신 N decorator.

## 3. Decorator vs Proxy vs Adapter

| | Decorator | Proxy | Adapter |
| --- | --- | --- | --- |
| 의도 | 책임 추가 | 대리 (제어 / lazy / remote) | 인터페이스 변환 |
| 같은 interface | ✓ | ✓ | ✗ (변환) |
| 다중 chain | 일반적 | 보통 1 단 | 1 단 |

## 4. 구조

```
Component (interface)
  + operation()
  ▲
  ├── ConcreteComponent     ← 기본
  │
  └── Decorator
      - inner: Component
      + operation() {
          // 추가 일
          inner.operation();
          // 추가 일
        }
```

## 5. Java

```java
interface DataReader {
    String read();
}

class FileReader implements DataReader {
    public String read() { return "raw"; }
}

abstract class Decorator implements DataReader {
    protected DataReader inner;
    Decorator(DataReader i) { this.inner = i; }
}

class CompressionDecorator extends Decorator {
    CompressionDecorator(DataReader i) { super(i); }
    public String read() { return decompress(inner.read()); }
    private String decompress(String s) { return s; /* ... */ }
}

class EncryptionDecorator extends Decorator {
    EncryptionDecorator(DataReader i) { super(i); }
    public String read() { return decrypt(inner.read()); }
    private String decrypt(String s) { return s; /* ... */ }
}

// 사용
DataReader r = new EncryptionDecorator(new CompressionDecorator(new FileReader()));
r.read();  // file → decompress → decrypt
```

**Java IO** = 표준 Decorator 예: `new BufferedReader(new InputStreamReader(new FileInputStream(...)))`.

**Spring `@Transactional` / `@Cacheable` / `@Async`**: AOP 가 메서드를 decorator 로 wrap.

## 6. Python — Function decorator

```python
import functools, time

def timing(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = fn(*args, **kwargs)
        print(f"{fn.__name__}: {time.time()-start:.3f}s")
        return result
    return wrapper

@timing
def slow_function():
    time.sleep(1)

slow_function()  # slow_function: 1.001s
```

- Python `@decorator` 문법 자체가 GoF Decorator.
- `functools.wraps` 로 metadata 보존.

**Chain**:
```python
@timing
@retry(times=3)
@cache
def fetch_user(id): ...
```

→ `fetch_user = timing(retry(cache(fetch_user)))`.

**Django**:
- `@login_required` / `@permission_required` view decorator.
- `@cache_page(60)` view caching.

**FastAPI**:
- `@app.get("/path")` 자체가 decorator (라우팅).
- Middleware: 함수 형태로 chain.

```python
@app.middleware("http")
async def add_timing(request: Request, call_next):
    start = time.time()
    response = await call_next(request)
    response.headers["X-Time"] = f"{time.time()-start:.3f}"
    return response
```

## 7. TypeScript — Class decorator + Method decorator

```ts
// Method decorator (ES proposal, NestJS / TypeORM 광범위)
function log(target: any, key: string, desc: PropertyDescriptor) {
  const orig = desc.value;
  desc.value = function(...args: any[]) {
    console.log(`call ${key}(${args})`);
    return orig.apply(this, args);
  };
}

class Service {
  @log
  doWork(x: number) { return x * 2; }
}
```

**NestJS**:
```ts
@Controller('users')
export class UserController {
  @Get(':id')
  @UseInterceptors(CacheInterceptor)
  @UseGuards(AuthGuard)
  async findOne(@Param('id') id: string) { /* ... */ }
}
```

— Controller / route / guard / interceptor / pipe 모두 decorator chain.

**TypeORM**:
```ts
@Entity()
class User {
  @PrimaryGeneratedColumn() id: number;
  @Column() name: string;
  @Index() @Unique() email: string;
}
```

**React HOC (Higher-Order Component)** = decorator pattern:
```tsx
function withLogging<P>(C: React.ComponentType<P>) {
  return (props: P) => {
    useEffect(() => { console.log('rendered'); });
    return <C {...props} />;
  };
}

const Wrapped = withLogging(MyComponent);
```

**Express middleware** = decorator chain:
```ts
app.use(cors());
app.use(helmet());
app.use(rateLimit({...}));
app.use(authMiddleware);
app.get('/', handler);
```

## 8. Go — middleware chain

```go
type Handler func(w http.ResponseWriter, r *http.Request)

func Logging(next Handler) Handler {
    return func(w http.ResponseWriter, r *http.Request) {
        log.Printf("%s %s", r.Method, r.URL.Path)
        next(w, r)
    }
}

func Auth(next Handler) Handler {
    return func(w http.ResponseWriter, r *http.Request) {
        if r.Header.Get("Authorization") == "" {
            http.Error(w, "unauthorized", 401)
            return
        }
        next(w, r)
    }
}

// chain
handler := Logging(Auth(actualHandler))
http.HandleFunc("/", handler)
```

**Go std `net/http`** = decorator chain.
**Gin / Echo / Chi** = middleware chain (decorator).

## 9. Decorator stack 순서 함정

```python
@A
@B
@C
def f(): ...
```

순서:
- `f = A(B(C(f)))`.
- A 의 wrapper 가 가장 바깥.
- 호출 순서: A entry → B entry → C entry → 실 f → C exit → B exit → A exit.

→ `@cache` 가 `@auth` 아래에 있으면 cache 가 auth 우회 (위험).

## 10. 함정

1. **Decorator chain 너무 깊음** — 디버깅 / 추적 어려움. stack trace 가 wrapper 로 가득.
2. **`functools.wraps` 누락** — `__name__` / `__doc__` / signature 잃음.
3. **decorator order 의 의미** — `@cache @auth` vs `@auth @cache` 는 다른 동작. 순서 명시 / 코멘트.
4. **Stateful decorator + global** — 함수 별 state 가 cache 처럼 의도된 거면 OK, 의도치 않으면 leak.
5. **AOP (Spring) 의 self-invocation 우회** — `this.method()` 호출은 proxy 우회. constructor injection 필요.
6. **TypeScript decorator의 stage** — TS 5.0 의 ES proposal decorator 가 옛 `experimentalDecorators` 와 다른 시그니처.

## 11. 관련

- [[../proxy/proxy]] — 비슷한 구조, 의도가 "제어".
- [[../adapter/adapter]] — 인터페이스 변환.
- [[../composite/composite]] — 같은 interface 의 tree.
- [[../chain-of-responsibility/chain-of-responsibility]] — middleware chain 의 다른 시각.
