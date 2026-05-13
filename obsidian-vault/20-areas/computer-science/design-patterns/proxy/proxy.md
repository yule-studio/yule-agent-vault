---
title: "Proxy"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, gof, structural, proxy, lazy, cache, remote]
---

# Proxy

**[[../design-patterns|↑ 디자인 패턴]]** · GoF Structural

## 1. 한 줄

**다른 객체의 대리 (대행).** 호출자가 진짜 객체와 같은 interface 로 사용. 중간에 lazy / cache / 권한 / remote / logging 등 제어.

## 2. Proxy 종류

| 종류 | 의도 |
| --- | --- |
| **Virtual Proxy** | 비싼 객체 lazy 생성 |
| **Cache Proxy** | 결과 캐시 |
| **Protection Proxy** | 접근 권한 검사 |
| **Remote Proxy** | 원격 객체 대리 (RPC client) |
| **Smart Reference** | reference count / 로깅 / 트랜잭션 |
| **Logging Proxy** | 호출 기록 |

## 3. Proxy vs Decorator vs Adapter

| | Proxy | Decorator | Adapter |
| --- | --- | --- | --- |
| 목적 | **제어 / 대리** | 책임 추가 | 인터페이스 변환 |
| 의도 | 호출 access control | 행동 확장 | 호환 |
| chain | 보통 1 단 | 다중 |  1 단 |

## 4. 구조

```
Client → Subject (interface)
              ▲
              ├── RealSubject
              └── Proxy
                  - real: RealSubject
                  + operation() {
                      // check, lazy load, cache, etc.
                      real.operation();
                    }
```

## 5. Java / Spring

### Virtual Proxy (lazy)

```java
interface Image {
    void display();
}

class HighResImage implements Image {
    private final String path;
    public HighResImage(String p) {
        this.path = p;
        loadFromDisk();  // 비쌈
    }
    private void loadFromDisk() { /* ... */ }
    public void display() { /* ... */ }
}

class ImageProxy implements Image {
    private final String path;
    private HighResImage real;
    public ImageProxy(String p) { this.path = p; }
    public void display() {
        if (real == null) real = new HighResImage(path);  // lazy
        real.display();
    }
}
```

### Spring AOP — Proxy 의 가장 큰 사용처

```java
@Service
public class OrderService {
    @Transactional               // ← Spring 이 proxy 생성
    @Cacheable("orders")          // ← proxy 가 cache check
    @PreAuthorize("hasRole('ADMIN')")  // ← proxy 가 권한 check
    public Order get(Long id) {
        return repository.findById(id);
    }
}
```

- Spring 이 런타임에 `OrderService` 의 proxy 인스턴스 만듦.
- caller 가 `service.get(id)` 호출 시 실은 proxy 가 받음 → transaction begin → cache check → security check → real method → cache put → transaction commit.

### JDK Dynamic Proxy

```java
import java.lang.reflect.*;

interface Greeting { String hello(); }

class RealGreeting implements Greeting {
    public String hello() { return "hi"; }
}

Greeting real = new RealGreeting();
Greeting proxy = (Greeting) Proxy.newProxyInstance(
    Greeting.class.getClassLoader(),
    new Class<?>[]{Greeting.class},
    (p, method, args) -> {
        System.out.println("before");
        Object result = method.invoke(real, args);
        System.out.println("after");
        return result;
    }
);
proxy.hello();
```

- 인터페이스만 알면 런타임 proxy 생성.
- Spring AOP / Mockito / JPA 의 lazy entity 가 사용.

### CGLIB — class proxy (인터페이스 없어도)

- Spring `@Component class Foo {}` 의 proxy 는 CGLIB.
- final class / final method 못 함.

## 6. Python / Django

```python
class HighResImage:
    def __init__(self, path):
        self.path = path
        self._load()      # 비쌈
    def _load(self): pass
    def display(self): pass

class ImageProxy:
    def __init__(self, path):
        self.path = path
        self._real = None
    def display(self):
        if self._real is None:
            self._real = HighResImage(self.path)
        self._real.display()
```

**`__getattr__` 활용** — 동적 proxy:

```python
class LoggingProxy:
    def __init__(self, target):
        self._target = target
    def __getattr__(self, name):
        attr = getattr(self._target, name)
        if callable(attr):
            def wrapper(*args, **kwargs):
                print(f"call {name}")
                return attr(*args, **kwargs)
            return wrapper
        return attr

p = LoggingProxy(some_object)
p.method()  # "call method" 로깅 + 실 호출
```

**Django ORM lazy QuerySet**:
```python
users = User.objects.filter(active=True)   # SQL 실행 안 됨 (proxy)
print(users.count())                       # 여기서 처음 SQL.
```

`QuerySet` 이 proxy — 평가 미루기.

**Django `@cache_page`** decorator = cache proxy.

**FastAPI middleware** = proxy chain.

## 7. TypeScript / Node / React

### ES6 Proxy

```ts
const target = { name: 'Alice', age: 30 };

const proxy = new Proxy(target, {
  get(obj, prop, receiver) {
    console.log(`get ${String(prop)}`);
    return Reflect.get(obj, prop, receiver);
  },
  set(obj, prop, value) {
    console.log(`set ${String(prop)} = ${value}`);
    return Reflect.set(obj, prop, value);
  },
});

proxy.name;      // "get name" + "Alice"
proxy.age = 31;  // "set age = 31"
```

- JavaScript 의 표준 Proxy — getter/setter/method 모두 hook.

### Vue 3 reactivity = Proxy

```ts
import { reactive } from 'vue';

const state = reactive({ count: 0 });
state.count++;   // Vue 가 자동 re-render
```

- Vue 3 의 reactivity 시스템이 Proxy 기반.

### MobX, Immer, Valtio 도 Proxy 활용.

### NestJS Guards / Interceptors

```ts
@Controller('users')
@UseGuards(AuthGuard)               // ← proxy 가 권한 check
@UseInterceptors(LoggingInterceptor) // ← proxy 가 logging
export class UserController { ... }
```

### React.lazy + Suspense

```tsx
const LazyComponent = React.lazy(() => import('./Heavy'));
// 실 import 는 첫 render 시. 그 전엔 proxy 상태.
```

## 8. Go

### Function wrapper as proxy

```go
type Greeting func() string

func realGreeting() string { return "hi" }

func loggingProxy(g Greeting) Greeting {
    return func() string {
        log.Println("before")
        result := g()
        log.Println("after")
        return result
    }
}

greet := loggingProxy(realGreeting)
greet()
```

### HTTP reverse proxy — Go std

```go
import "net/http/httputil"

target, _ := url.Parse("http://backend:8080")
proxy := httputil.NewSingleHostReverseProxy(target)

http.HandleFunc("/", proxy.ServeHTTP)
```

- Go std `net/http/httputil` 의 ReverseProxy = full Remote Proxy.

### gRPC client = Remote Proxy

```go
conn, _ := grpc.Dial("server:50051", grpc.WithInsecure())
client := pb.NewServiceClient(conn)

resp, err := client.SayHello(ctx, &pb.HelloReq{Name: "world"})
// client 가 grpc stub — Remote Proxy.
```

## 9. ORM 의 lazy loading (가장 큰 실 사용 사례)

```java
@Entity
class User {
    @OneToMany(fetch = FetchType.LAZY)
    List<Order> orders;
}

User u = repo.findById(1);
// u.getOrders() 호출 시점에 SQL — Hibernate proxy.
```

- Hibernate 의 lazy collection 이 Proxy.
- 같은 idea: Django `select_related` / `prefetch_related` 없이 lazy access.

**함정**: lazy load + transaction 닫힌 후 access = `LazyInitializationException`. `OPEN_SESSION_IN_VIEW` 또는 explicit fetch.

## 10. 함정

1. **Proxy 의 self-invocation 우회** (Spring) — class 내부에서 `this.transactionalMethod()` 호출 시 proxy 안 거침. constructor injection 또는 별도 service.
2. **lazy proxy 의 N+1 query** — list 의 각 item 의 lazy field access = N 개 query. eager fetch / batch.
3. **proxy chain 너무 깊음** — debugging hell.
4. **proxy 의 equality 함정** — `proxy.equals(real)` 가 false 일 수 있음.
5. **JDK dynamic proxy 는 interface 만** — class proxy 는 CGLIB / ByteBuddy.
6. **Vue Proxy 의 reactivity** — Date / Map / Set 같은 일부 native 객체는 별도 처리.
7. **JavaScript Proxy 의 unsupported trap** — 모든 operation hook 안 됨. spec 확인.

## 11. 관련

- [[../decorator/decorator]] — 책임 추가.
- [[../adapter/adapter]] — 인터페이스 변환.
- [[../facade/facade]] — subsystem 단순화.
- [[../chain-of-responsibility/chain-of-responsibility]] — middleware chain.
