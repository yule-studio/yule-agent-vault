---
title: "Singleton — Go"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, gof, creational, singleton, go, sync-once]
---

# Singleton — Go

**[[singleton|↑ Singleton hub]]**

## 1. Package-level 변수 + `sync.Once`

```go
package logger

import (
    "log"
    "sync"
)

type Logger struct {
    *log.Logger
}

var (
    instance *Logger
    once     sync.Once
)

func Get() *Logger {
    once.Do(func() {
        instance = &Logger{log.Default()}
    })
    return instance
}

func (l *Logger) Log(msg string) {
    l.Println(msg)
}
```

```go
// 사용
logger.Get().Log("hello")
```

- `sync.Once` 가 `Do` 의 함수를 정확히 1 번 실행 (thread-safe).
- Go 의 표준 Singleton 패턴.

## 2. Eager — package init

```go
package logger

import "log"

type Logger struct {
    *log.Logger
}

var instance = &Logger{log.Default()}

func Get() *Logger {
    return instance
}
```

- Go 의 `init()` / 변수 초기화는 1 번만.
- thread-safe 자동.
- lazy 가 필요 없으면 가장 단순.

## 3. Package 자체가 Singleton

```go
package logger

import "log"

var level = "INFO"

func SetLevel(l string) { level = l }

func Log(msg string) {
    log.Printf("[%s] %s", level, msg)
}
```

```go
// 사용
logger.Log("hello")
logger.SetLevel("DEBUG")
```

- 인스턴스 노출 없음.
- Go community 가장 idiomatic.
- Python module-level singleton 과 같은 사상.

## 4. Database / Connection Pool — `*sql.DB`

```go
package db

import (
    "database/sql"
    "sync"
    _ "github.com/lib/pq"
)

var (
    pool *sql.DB
    once sync.Once
)

func Pool() *sql.DB {
    once.Do(func() {
        var err error
        pool, err = sql.Open("postgres", os.Getenv("DATABASE_URL"))
        if err != nil { panic(err) }
        pool.SetMaxOpenConns(50)
    })
    return pool
}
```

- `*sql.DB` 자체가 connection pool.
- process 에 1 개만 권장 (Go std doc 명시).

## 5. Configuration Singleton

```go
package config

import (
    "encoding/json"
    "os"
    "sync"
)

type Config struct {
    DatabaseURL string `json:"database_url"`
    LogLevel    string `json:"log_level"`
}

var (
    cfg  *Config
    once sync.Once
)

func Load() *Config {
    once.Do(func() {
        f, _ := os.Open("config.json")
        defer f.Close()
        cfg = &Config{}
        json.NewDecoder(f).Decode(cfg)
    })
    return cfg
}
```

## 6. HTTP server / handler 에서

```go
package main

import (
    "net/http"
    "yourapp/logger"
    "yourapp/db"
)

func handler(w http.ResponseWriter, r *http.Request) {
    logger.Log("request received")
    rows, _ := db.Pool().Query("SELECT 1")
    defer rows.Close()
    // ...
}

func main() {
    http.HandleFunc("/", handler)
    http.ListenAndServe(":8080", nil)
}
```

- handler 가 매번 호출돼도 `logger.Log` 와 `db.Pool()` 은 같은 인스턴스.
- net/http 의 ServeMux 도 singleton.

## 7. Go 에서 Singleton 이 자연스러운 이유

1. **goroutine 가 자유로움** — 같은 인스턴스 공유 + 적절한 lock 으로 동시 사용.
2. **`sync.Once` 라는 표준 도구** — race-free lazy init.
3. **package = namespace** — 외부에 인스턴스 노출 없이 함수만 export.
4. **Interface 가 작아 합성 친화** — Singleton 도 interface 통해 mock 가능.

## 8. 테스트 — interface + DI

Singleton 의 테스트 문제를 Go 답게 해결:

```go
// logger.go
type Logger interface {
    Log(msg string)
}

type realLogger struct{}
func (r *realLogger) Log(msg string) { fmt.Println(msg) }

// app.go
type App struct {
    logger Logger      // interface 받음
}

func NewApp(l Logger) *App { return &App{logger: l} }

func (a *App) DoWork() {
    a.logger.Log("working")
}

// production
app := NewApp(&realLogger{})

// test
type fakeLogger struct {
    msgs []string
}
func (f *fakeLogger) Log(msg string) { f.msgs = append(f.msgs, msg) }

func TestApp(t *testing.T) {
    fake := &fakeLogger{}
    app := NewApp(fake)
    app.DoWork()
    if len(fake.msgs) != 1 { t.Fail() }
}
```

→ Go 의 interface 작음 + 명시적 의존성 주입이 Singleton 의 테스트 문제 해결.

## 9. context.Context — Singleton 의 대안

```go
// 전체 lifecycle 동안 1 개 logger 가 필요하면 ctx 에 담아 전달
type ctxKey string
const loggerKey ctxKey = "logger"

func WithLogger(ctx context.Context, l Logger) context.Context {
    return context.WithValue(ctx, loggerKey, l)
}

func FromContext(ctx context.Context) Logger {
    return ctx.Value(loggerKey).(Logger)
}

// 사용
ctx := WithLogger(context.Background(), &realLogger{})
FromContext(ctx).Log("hi")
```

- 전역 변수 대신 ctx 로 전달.
- request-scoped logger / tracer 에 좋음.

## 10. gRPC / web framework — server 별 singleton

```go
// gRPC
import "google.golang.org/grpc"

var server = grpc.NewServer()    // package 1 개

// 등록
RegisterServiceServer(server, myImpl)
server.Serve(lis)
```

- 한 binary 의 server 1 개.
- multiple goroutine 이 동시 RPC 처리.

## 11. Concurrency 패턴 — Singleton 안의 lock

```go
type Cache struct {
    mu    sync.RWMutex
    items map[string]string
}

var (
    cacheInstance *Cache
    cacheOnce     sync.Once
)

func GetCache() *Cache {
    cacheOnce.Do(func() {
        cacheInstance = &Cache{items: make(map[string]string)}
    })
    return cacheInstance
}

func (c *Cache) Get(k string) string {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.items[k]
}

func (c *Cache) Set(k, v string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.items[k] = v
}
```

- Singleton + RWMutex = 동시 read 가능.
- `sync.Map` 이 더 효율적인 경우 있음.

## 12. 함정 (Go specific)

1. **Init 순서 의존** — package init 순서가 import 순서 + alphabetical. 다른 package 의 init 에 의존하면 fragile.
2. **`sync.Once` 의 panic** — `Do` 의 함수가 panic 하면 `Once` 가 "이미 실행됨" 으로 표시. 재시도 안 됨.
3. **Mutex 의 deadlock** — Singleton method 안에서 다른 Singleton 의 method 호출 시 lock 순서 주의.
4. **Goroutine leak** — Singleton 의 background goroutine 이 cleanup 안 됨. `context` + `Close()` 권장.
5. **Test 의 race** — package-level 변수가 test 사이 state leak. `t.Cleanup()` 또는 별도 `Reset()`.
6. **Multi-binary 환경** — 같은 module 을 여러 binary 로 빌드 시 각 binary 별 singleton. shared state 필요면 Redis / DB.

## 13. 권장 — Go idiomatic 답

1. **단순 상태** = package-level 변수 + `init()`.
2. **Lazy 초기화** = `sync.Once`.
3. **DB pool / HTTP client** = package singleton with `sync.Once`.
4. **테스트 친화** = interface + DI.
5. **Request-scoped** = `context.Context` value.

→ Go 에는 강요된 OOP Singleton 이 없음. `sync.Once` + package 가 충분.

## 14. 관련

- [[singleton]]
- [[singleton-java]]
- [[singleton-python]]
- [[singleton-typescript]]
