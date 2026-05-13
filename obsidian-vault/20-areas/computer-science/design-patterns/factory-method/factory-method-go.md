---
title: "Factory Method — Go"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, factory-method, go]
---

# Factory Method — Go

**[[factory-method|↑ Factory Method hub]]**

## 1. 기본 — `New<X>` 함수

```go
package storage

type Storage interface {
    Save(key, value string) error
}

type localStorage struct{ /* ... */ }
func (l *localStorage) Save(k, v string) error { /* ... */ }

type s3Storage struct{ bucket string }
func (s *s3Storage) Save(k, v string) error { /* ... */ }

// factory functions
func NewLocal() Storage {
    return &localStorage{}
}

func NewS3(bucket string) Storage {
    return &s3Storage{bucket: bucket}
}
```

- Go 의 표준 idiom: `NewXxx` 가 factory function.
- 인터페이스 반환 → 호출자가 구체 타입 모름.

## 2. `database/sql.Open` — 표준 Go factory

```go
import (
    "database/sql"
    _ "github.com/lib/pq"            // driver 등록
    _ "github.com/go-sql-driver/mysql"
)

db, err := sql.Open("postgres", "postgres://...")
// driver name 으로 어떤 구현 만들지 결정
```

- `_ "github.com/lib/pq"` 의 init() 가 driver 등록.
- `sql.Open(name, ...)` 이 registry 에서 driver 찾아 connection 만듦.

## 3. Driver registration pattern

```go
package storage

import "sync"

type Driver interface {
    New(config string) (Storage, error)
}

var (
    drivers   = make(map[string]Driver)
    driversMu sync.RWMutex
)

func Register(name string, d Driver) {
    driversMu.Lock()
    defer driversMu.Unlock()
    drivers[name] = d
}

func Open(name, config string) (Storage, error) {
    driversMu.RLock()
    d, ok := drivers[name]
    driversMu.RUnlock()
    if !ok { return nil, fmt.Errorf("unknown: %s", name) }
    return d.New(config)
}
```

```go
// driver 패키지
package s3

import "yourapp/storage"

type s3Driver struct{}
func (s3Driver) New(config string) (storage.Storage, error) { /* ... */ }

func init() {
    storage.Register("s3", s3Driver{})
}

// main 에서
import _ "yourapp/storage/s3"   // init() 가 등록

s, _ := storage.Open("s3", "bucket=mybucket")
```

- `database/sql` / `image` (image format decoder) 가 정확히 이 패턴.

## 4. Options pattern + factory

```go
type ServerOption func(*Server)

func WithPort(p int) ServerOption {
    return func(s *Server) { s.port = p }
}

func WithTLS(cert, key string) ServerOption {
    return func(s *Server) { s.tls = &tlsConfig{cert, key} }
}

func NewServer(opts ...ServerOption) *Server {
    s := &Server{port: 8080}     // default
    for _, opt := range opts {
        opt(s)
    }
    return s
}

// 사용
srv := NewServer(WithPort(9000), WithTLS("cert.pem", "key.pem"))
```

- Go 의 가장 idiomatic factory + builder 결합.
- gRPC, kubernetes client, AWS SDK 가 광범위 사용.

## 5. `http.NewServeMux` — std factory

```go
mux := http.NewServeMux()
mux.HandleFunc("/", handler)

server := &http.Server{
    Addr:    ":8080",
    Handler: mux,
}
```

## 6. Concrete factory function for tests

```go
// production
func NewRepo(db *sql.DB) Repo {
    return &postgresRepo{db}
}

// test
func newTestRepo() Repo {
    return &fakeRepo{
        users: map[int]User{1: {Name: "Alice"}},
    }
}
```

- Go 의 test 에서 별도 factory function 흔함.
- DI 없이도 충분.

## 7. Generic factory (Go 1.18+)

```go
type Constructor[T any] func() T

func Make[T any](c Constructor[T]) T {
    return c()
}

type Cache[T any] struct { items map[string]T }

func NewCache[T any]() *Cache[T] {
    return &Cache[T]{items: make(map[string]T)}
}

// 사용
c := NewCache[User]()
```

## 8. Interface 분리 — production vs test

```go
type Cache interface {
    Get(k string) (string, bool)
    Set(k, v string)
}

// real
type redisCache struct{ rdb *redis.Client }
func NewRedisCache(addr string) Cache { return &redisCache{redis.NewClient(...)} }

// fake (test)
type memCache struct{ m map[string]string }
func NewMemCache() Cache { return &memCache{m: map[string]string{}} }
```

## 9. 함정

1. **factory 가 거대 switch / if-else** — driver registration 으로.
2. **`init()` 의 등록 순서 의존** — package init 순서가 alphabetical + import 순서.
3. **`sync.Map` 의 type unsafe** — driver registry 에 generic 권장 (Go 1.18+).
4. **interface 미공개 + 구체 type return** — 호출자가 구체 타입 알게 됨. interface return 권장.
5. **options pattern 의 default 누락** — `NewX()` 가 zero-value 안 됨. 명시적 default 세팅.
6. **factory 가 lock 없이 mutable state** — registry 등은 sync.Mutex.

## 10. 관련

- [[factory-method]]
- [[factory-method-java]]
- [[factory-method-python]]
- [[factory-method-typescript]]
