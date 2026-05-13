---
title: "Singleton — Java / Spring"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, gof, creational, singleton, java, spring, jakarta]
---

# Singleton — Java / Spring

**[[singleton|↑ Singleton hub]]**

## 1. 기본 — Eager 초기화

```java
public class Logger {
    private static final Logger INSTANCE = new Logger();
    private Logger() {}
    public static Logger getInstance() { return INSTANCE; }
    public void log(String msg) { /* ... */ }
}
```

- class loader 가 시점에 한 번만 초기화.
- thread-safe (JVM 보장).
- lazy 아님.

## 2. Lazy + Thread-safe (Bill Pugh — Initialization-on-demand holder)

```java
public class Logger {
    private Logger() {}
    private static class Holder {
        static final Logger INSTANCE = new Logger();
    }
    public static Logger getInstance() { return Holder.INSTANCE; }
}
```

- Holder 가 처음 참조될 때만 초기화.
- JVM 의 class init 락이 thread-safe 보장.
- **Java 권장 idiom**.

## 3. Enum Singleton (Joshua Bloch 권장)

```java
public enum Logger {
    INSTANCE;
    public void log(String msg) { /* ... */ }
}

// 사용
Logger.INSTANCE.log("hello");
```

- enum 의 단일 인스턴스 보장은 JVM 차원.
- Reflection / Serialization 우회 자동 차단.
- 가장 안전한 Java Singleton.

## 4. Double-Checked Locking (성능 최적화)

```java
public class Logger {
    private static volatile Logger instance;
    private Logger() {}
    public static Logger getInstance() {
        if (instance == null) {                 // 1차 check (lock 없이)
            synchronized (Logger.class) {
                if (instance == null) {         // 2차 check
                    instance = new Logger();
                }
            }
        }
        return instance;
    }
}
```

- `volatile` 필수 — 안 쓰면 partial-construct 객체 보일 위험.
- Java 5+ 의 memory model 에서 volatile 가 happens-before 보장.

## 5. Spring 의 Bean — 기본 Singleton

```java
@Component
public class LoggerService {
    public void log(String msg) { /* ... */ }
}

@Service
public class OrderService {
    private final LoggerService logger;

    public OrderService(LoggerService logger) {  // DI
        this.logger = logger;
    }
}
```

- Spring `ApplicationContext` 가 `@Component` / `@Service` / `@Repository` 의 default scope = **singleton**.
- 컨테이너가 인스턴스 1 개 관리 + 다른 bean 에 inject.
- 명시적 코드 0 으로 Singleton 효과.

### Scope 명시

```java
@Component
@Scope("singleton")        // default
@Scope("prototype")        // 매번 새 인스턴스
@Scope("request")          // HTTP 요청 별
@Scope("session")          // HTTP session 별
```

- web app 의 일부 bean = request/session scope 가 더 맞음.

## 6. Jakarta EE / CDI

```java
@ApplicationScoped         // CDI 의 Singleton 동급
public class LoggerService {
    public void log(String msg) { /* ... */ }
}

@Singleton                 // EJB 의 Singleton (트랜잭션 / lifecycle 지원)
public class CacheManager {
    @PostConstruct
    void init() { /* ... */ }
}
```

## 7. Spring 의 prototype 안에서 singleton 주입의 함정

```java
@Component
@Scope("prototype")
public class TaskWorker {
    @Autowired private OrderService orderService;  // singleton bean
}

@Component  // singleton (default)
public class Coordinator {
    @Autowired private TaskWorker worker;  // ← 항상 같은 worker 반복!
}
```

- singleton bean 에 prototype bean 주입하면 `worker` 가 한 번만 생성됨.
- 매번 새 인스턴스가 필요하면 **`ObjectProvider<TaskWorker>`** 또는 **`@Lookup`** 사용.

```java
@Component
public class Coordinator {
    @Autowired private ObjectProvider<TaskWorker> workerProvider;

    public void runTask() {
        TaskWorker w = workerProvider.getObject();  // 매번 새 인스턴스
    }
}
```

## 8. Testing — Singleton 의 가장 큰 문제

### 옛 (static getInstance)

```java
@Test
void testOrder() {
    Logger.getInstance().log("...");   // static — mock 불가
}
```

- mock 불가 → integration test 만 가능.

### Spring DI 로

```java
@SpringBootTest
class OrderServiceTest {
    @MockBean
    LoggerService logger;             // ← 자동 mock

    @Autowired
    OrderService service;

    @Test
    void testOrder() {
        service.placeOrder(...);
        verify(logger).log(anyString());
    }
}
```

→ **DI 가 Singleton 의 테스트 문제를 거의 해결**.

## 9. 함정 (Java specific)

1. **Class loader 별 singleton** — Java EE app server 의 multi-classloader 환경에서 인스턴스 N 개.
2. **Reflection 우회** — `Class.forName("Logger").getDeclaredConstructor().setAccessible(true).newInstance()` 로 추가 인스턴스 가능. Enum 만 안 통함.
3. **Serialization 우회** — Singleton class 가 `Serializable` 이면 deserialize 시 새 인스턴스. `readResolve()` 로 막아야.
4. **Lazy 초기화 시 race** — synchronized 없이 lazy 만들면 race. Holder idiom 권장.
5. **Spring prototype 의 inject 함정** — §7 참조.
6. **GraalVM native image** — static 초기화가 build-time vs run-time. `@RegisterReflectionForBinding` 등 주의.

## 10. 관련

- [[singleton]]
- [[singleton-python]]
- [[singleton-typescript]]
- [[singleton-go]]
