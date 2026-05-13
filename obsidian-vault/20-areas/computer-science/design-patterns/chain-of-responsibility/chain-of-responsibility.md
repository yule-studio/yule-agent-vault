---
title: "Chain of Responsibility"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, gof, behavioral, chain-of-responsibility, middleware, filter]
---

# Chain of Responsibility

**[[../design-patterns|↑ 디자인 패턴]]** · GoF Behavioral

## 1. 한 줄

**요청을 처리자 체인에 따라 넘긴다. 누군가 처리하거나 끝까지 안 처리되면 통과.**

## 2. 언제 쓰는가

- **HTTP middleware pipeline** — auth / logging / rate limit / CORS / handler.
- **이벤트 버블링** — DOM, GUI.
- **권한 escalation** — 매니저 → 부장 → 임원 승인.
- **filter chain** — Spring Security 의 SecurityFilterChain.
- **command pipeline** — Servlet filter, gRPC interceptor.

## 3. 구조

```
Handler (interface)
  - next: Handler
  + setNext(h)
  + handle(request)
  ▲
  └── ConcreteHandler { handle(req) { if can { do }; else next?.handle(req) } }

Client → handler1 → handler2 → handler3 → ...
```

## 4. Java / Spring

```java
abstract class AuthHandler {
    protected AuthHandler next;
    AuthHandler setNext(AuthHandler n) { this.next = n; return n; }
    abstract Result handle(Request r);
}

class RoleCheck extends AuthHandler {
    Result handle(Request r) {
        if (!r.user.hasRole("USER")) return Result.deny("role");
        return next != null ? next.handle(r) : Result.ok();
    }
}

class RateLimit extends AuthHandler {
    Result handle(Request r) {
        if (r.requestsPerMin > 100) return Result.deny("rate");
        return next != null ? next.handle(r) : Result.ok();
    }
}

// chain
AuthHandler chain = new RoleCheck();
chain.setNext(new RateLimit());
chain.handle(req);
```

**Spring Security FilterChain** = chain of responsibility의 산업 표준:
```java
http
  .csrf().disable()
  .authorizeHttpRequests(a -> a
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .anyRequest().authenticated())
  .formLogin(...)
  .oauth2Login(...);
// 내부적으로 여러 Filter 가 chain 으로 실행
```

**Servlet Filter** / **Spring Interceptor** / **JAX-RS Filter** 모두 같은 패턴.

## 5. Python / Django / FastAPI

```python
from typing import Optional, Callable

class Handler:
    def __init__(self):
        self._next: Optional["Handler"] = None
    def set_next(self, h): self._next = h; return h
    def handle(self, request) -> Optional[str]:
        if self._next:
            return self._next.handle(request)
        return None

class AuthHandler(Handler):
    def handle(self, request):
        if not request.get("user"):
            return "deny: no user"
        return super().handle(request)

class QuotaHandler(Handler):
    def handle(self, request):
        if request.get("quota_used", 0) > 100:
            return "deny: quota"
        return super().handle(request)

chain = AuthHandler()
chain.set_next(QuotaHandler())
result = chain.handle({"user": "alice", "quota_used": 50})
```

**Django Middleware** = chain:
```python
# settings.py
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'myapp.middleware.CustomMiddleware',
]
```
- 각 middleware 가 request 처리 후 다음으로 → view → 다시 거꾸로 response 변환.

**FastAPI middleware**:
```python
@app.middleware("http")
async def custom_middleware(request, call_next):
    # before
    response = await call_next(request)
    # after
    return response
```

## 6. TypeScript / Node / NestJS / React

```ts
abstract class Handler {
  protected next?: Handler;
  setNext(h: Handler) { this.next = h; return h; }
  handle(request: Request): Response | null {
    return this.next?.handle(request) ?? null;
  }
}

class AuthHandler extends Handler {
  handle(req: Request) {
    if (!req.user) return { status: 401, body: 'unauthorized' };
    return super.handle(req);
  }
}

class RateLimitHandler extends Handler {
  handle(req: Request) {
    if (req.rate > 100) return { status: 429, body: 'rate limit' };
    return super.handle(req);
  }
}

const chain = new AuthHandler();
chain.setNext(new RateLimitHandler());
```

**Express middleware** = chain of responsibility의 자바스크립트 산업 표준:
```ts
app.use(cors());
app.use(helmet());
app.use(rateLimit({...}));
app.use(authMiddleware);
app.get('/', handler);
```

**NestJS Guards / Interceptors / Pipes / Exception Filters** = 각각 별도의 chain.

**Redux middleware** = chain:
```ts
const store = createStore(
  reducer,
  applyMiddleware(logger, thunk, errorReporter)  // chain
);
```

**React event bubbling** = DOM 의 chain.

## 7. Go

```go
type Handler interface {
    Handle(r Request) error
    SetNext(Handler)
}

type baseHandler struct{ next Handler }
func (b *baseHandler) SetNext(h Handler) { b.next = h }
func (b *baseHandler) callNext(r Request) error {
    if b.next != nil { return b.next.Handle(r) }
    return nil
}

type AuthHandler struct{ baseHandler }
func (a *AuthHandler) Handle(r Request) error {
    if r.User == "" { return errors.New("unauthorized") }
    return a.callNext(r)
}
```

**Go `net/http` middleware** = chain (function chain):
```go
func authMW(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.Header.Get("Authorization") == "" {
            http.Error(w, "unauthorized", 401)
            return
        }
        next.ServeHTTP(w, r)
    })
}

handler := authMW(loggingMW(rateLimitMW(actualHandler)))
http.Handle("/", handler)
```

**gRPC interceptor** = chain:
```go
grpc.NewServer(
    grpc.ChainUnaryInterceptor(
        loggingInterceptor,
        authInterceptor,
        rateLimitInterceptor,
    ),
)
```

**Gin / Echo / Chi** middleware = chain.

## 8. CoR vs Decorator vs Pipeline

| | CoR | Decorator | Pipeline |
| --- | --- | --- | --- |
| 모든 handler 실행 | ✗ (조건부) | ✓ | ✓ |
| 처리 후 다음으로 | 선택적 | 자동 | 자동 |
| 결과 합성 | 보통 X | ✓ | ✓ |
| 예 | filter / auth | logging wrap | data transform |

## 9. 함정

1. **handler 가 next 안 호출** — 의도된 stop 인지 bug 인지 모호. 명시적 return 값.
2. **무한 loop** — `setNext(self)` 같은 실수.
3. **순서 의존** — middleware 순서가 잘못되면 동작 다름. e.g. CORS 가 auth 후 면 preflight fail.
4. **handler 가 mutable shared state** — chain 마다 state pollution.
5. **error propagation** — 한 handler 의 exception 이 chain 의 다음 정상 흐름 막음. error wrap.
6. **dynamic chain build** — 런타임 chain 구성 시 thread-safety.

## 10. 관련

- [[../decorator/decorator]] — 모든 wrapper 실행.
- [[../command/command]] — 요청 객체화.
- [[../composite/composite]] — 트리 vs 선형 chain.
