---
title: "Builder"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, gof, creational, builder]
---

# Builder

**[[../design-patterns|↑ 디자인 패턴]]** · GoF Creational

## 1. 한 줄

**복잡한 객체를 단계별로 구성.** 매개변수가 많거나 일부만 선택적일 때.

## 2. 언제 쓰는가

- **매개변수가 많고 일부는 선택적** — HTTP request, SQL query, 거대 form.
- **불변 객체 (immutable)** — 모든 필드 final, builder 가 한 번에 채움.
- **단계별 검증** — `Cart.checkout()` 호출 전 단계마다 valid 확인.
- **DSL-like API** — 메서드 체이닝.

## 3. 안티 — Builder 가 필요 없는 경우

- 매개변수 3 개 이하 = 그냥 constructor.
- 객체가 mutable 이고 setter 충분.
- 매개변수가 안정 (변동 없음).

## 4. 구조

```
Director (선택적)  ──►  Builder
                          │
                          ▼
                      Product
```

- **Director**: 빌드 순서 고정 (있어도 되고 없어도 됨).
- **Builder**: 단계별 메서드 + `build()`.
- **Product**: 최종 객체.

## 5. Java — Lombok / 명시적

```java
// 명시적
public class HttpRequest {
    private final String url;
    private final String method;
    private final Map<String, String> headers;

    private HttpRequest(Builder b) {
        this.url = b.url;
        this.method = b.method;
        this.headers = b.headers;
    }

    public static Builder builder() { return new Builder(); }

    public static class Builder {
        private String url;
        private String method = "GET";
        private Map<String, String> headers = new HashMap<>();

        public Builder url(String u) { this.url = u; return this; }
        public Builder method(String m) { this.method = m; return this; }
        public Builder header(String k, String v) { this.headers.put(k, v); return this; }
        public HttpRequest build() {
            Objects.requireNonNull(url, "url");
            return new HttpRequest(this);
        }
    }
}

// 사용
HttpRequest req = HttpRequest.builder()
    .url("https://api.com")
    .method("POST")
    .header("Authorization", "Bearer ...")
    .build();
```

**Lombok 으로 단순화**:
```java
@Builder
public class HttpRequest {
    private String url;
    private String method;
    private Map<String, String> headers;
}
```

**Spring 예**: `WebClient.builder().baseUrl(...).filter(...).build()`. `MockMvc`, `RestTemplateBuilder` 모두 builder.

## 6. Python — `@dataclass` + `**kwargs` / 명시적

```python
from dataclasses import dataclass, field

@dataclass
class HttpRequest:
    url: str
    method: str = "GET"
    headers: dict = field(default_factory=dict)

# 가장 pythonic — keyword arguments
req = HttpRequest(url="https://api.com", method="POST", headers={"Authorization": "Bearer ..."})
```

**명시적 builder**:
```python
class HttpRequestBuilder:
    def __init__(self):
        self._url = None
        self._method = "GET"
        self._headers = {}

    def url(self, u): self._url = u; return self
    def method(self, m): self._method = m; return self
    def header(self, k, v): self._headers[k] = v; return self
    def build(self):
        if not self._url: raise ValueError("url required")
        return HttpRequest(url=self._url, method=self._method, headers=self._headers)

req = HttpRequestBuilder().url("...").method("POST").header("Auth", "...").build()
```

→ Python 의 keyword args + dataclass / Pydantic 이 보통 builder 대체.

**SQLAlchemy 의 Query** = builder의 완벽한 예: `session.query(User).filter(...).order_by(...).limit(10).all()`.

**Django QuerySet**: `User.objects.filter(...).exclude(...).order_by(...).values(...)`.

## 7. TypeScript / NestJS / React

```ts
class HttpRequestBuilder {
  private url?: string;
  private method = 'GET';
  private headers: Record<string, string> = {};

  setUrl(u: string) { this.url = u; return this; }
  setMethod(m: string) { this.method = m; return this; }
  addHeader(k: string, v: string) { this.headers[k] = v; return this; }
  build() {
    if (!this.url) throw new Error('url required');
    return { url: this.url, method: this.method, headers: this.headers };
  }
}

const req = new HttpRequestBuilder()
  .setUrl('https://api.com')
  .setMethod('POST')
  .addHeader('Auth', 'Bearer ...')
  .build();
```

**fluent API 의 대표 예**:
- **Axios**: `axios.create({...}).get(...)`.
- **Prisma**: `prisma.user.findMany({ where: {...}, include: {...} })`.
- **TanStack Query**: `useQuery({ queryKey: ..., queryFn: ... })`.
- **NestJS TypeORM QueryBuilder**: `repo.createQueryBuilder('u').where(...).leftJoin(...).getMany()`.

**React Form Builder** — react-hook-form 도 본질적으로 builder pattern (`register`, `handleSubmit`, `formState`).

## 8. Go — Functional options pattern (가장 idiomatic)

```go
type Server struct {
    addr    string
    timeout time.Duration
    tls     *tls.Config
}

type ServerOption func(*Server)

func WithAddr(a string) ServerOption { return func(s *Server) { s.addr = a } }
func WithTimeout(t time.Duration) ServerOption { return func(s *Server) { s.timeout = t } }
func WithTLS(c *tls.Config) ServerOption { return func(s *Server) { s.tls = c } }

func NewServer(opts ...ServerOption) *Server {
    s := &Server{
        addr:    ":8080",       // default
        timeout: 30 * time.Second,
    }
    for _, opt := range opts {
        opt(s)
    }
    return s
}

// 사용
srv := NewServer(
    WithAddr(":9000"),
    WithTimeout(60 * time.Second),
)
```

- gRPC, kubernetes client, AWS SDK 의 표준 API.
- builder 의 method chaining 대신 variadic option function.

**Builder struct 형태**도 있음:
```go
type RequestBuilder struct{ req *http.Request }
func (b *RequestBuilder) Method(m string) *RequestBuilder { b.req.Method = m; return b }
func (b *RequestBuilder) Build() *http.Request { return b.req }
```

## 9. 함정

1. **Telescoping constructor 회피 목적인데 builder 가 더 복잡** — 매개변수 < 4 면 builder 안 만들기.
2. **불변 보장 부족** — Builder 에서 build() 후에도 setter 호출 가능하면 product 가 mutable.
3. **partial state** — build() 호출 전 일부 필드 빠짐. `Objects.requireNonNull` / validation.
4. **race in concurrent builder** — Go 의 builder struct 가 goroutine 사이 공유 시 race.
5. **Director 의 over-engineering** — 빌드 순서 변동 없으면 Director 불필요.

## 10. 관련

- [[../factory-method/factory-method]] — 단순 객체.
- [[../abstract-factory/abstract-factory]] — 객체 가족.
- [[../prototype/prototype]] — 복제.
- [[../singleton/singleton]] — Builder 가 자체로 stateless 면 singleton 가능.
