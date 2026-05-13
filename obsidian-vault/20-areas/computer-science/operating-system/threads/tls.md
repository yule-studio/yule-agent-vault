---
title: "Thread Local Storage (TLS)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T09:50:00+09:00
tags:
  - operating-system
  - thread
  - tls
---

# Thread Local Storage

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | TLS 개념 + 언어별 |

**[[threads|↑ Threads hub]]**

---

## 1. 한 줄

같은 변수 이름이 **스레드마다 다른 인스턴스** 를 가짐.
락 없이 안전 — 공유 X.

---

## 2. 언어별

### 2.1 C / C++

```c
__thread int counter;             // GCC / Clang
thread_local int counter;          // C11 / C++11
```

```cpp
thread_local std::string buffer;
```

### 2.2 POSIX (런타임 키)

```c
pthread_key_t key;
pthread_key_create(&key, free_destructor);
pthread_setspecific(key, ptr);
void *p = pthread_getspecific(key);
```

destructor — 스레드 종료 시 자동 호출.

### 2.3 Java

```java
static final ThreadLocal<DateFormat> df =
    ThreadLocal.withInitial(() -> new SimpleDateFormat());

df.get().format(date);
df.remove();        // 누수 방지
```

### 2.4 Python

```python
import threading
local = threading.local()
local.value = 42      # 스레드별
```

### 2.5 Go
없음 — goroutine 은 stack 위주. context.Context 가 비슷한 역할.

### 2.6 Rust

```rust
thread_local! {
    static FOO: RefCell<u32> = RefCell::new(0);
}
FOO.with(|x| *x.borrow_mut() += 1);
```

---

## 3. 어디에 쓰나

- **non-thread-safe 라이브러리** 의 스레드별 인스턴스
  - `SimpleDateFormat` (Java), `strtok` (C)
- **per-thread cache / buffer**
  - 큰 buffer 재사용
- **request context**
  - 웹 서버의 current user / trace ID
- **lock-free counter** — 각자 카운트 후 합산

---

## 4. 구현 (개념)

```
컴파일러가 __thread 변수에 특수 모드 (TLS section)
런타임이 스레드 마다 TLS 블록 할당
변수 접근 시 [GS:offset] (x86-64) 같은 segment register 사용
```

→ 매우 빠름 (1-2 명령).

---

## 5. Pool + TLS 의 함정 ⚠️

```java
ThreadLocal<Connection> conn = ThreadLocal.withInitial(this::newConn);

// 풀 worker 가 task A 실행 → conn 생성
// task A 끝나도 conn 살아 있음
// task B 가 같은 worker 에서 실행 → 옛 conn 사용 → 누수 / 인증 정보 누수
```

해결:
```java
try {
    // do work
} finally {
    conn.remove();
}
```

또는 `try-with-resources` + scoped lifetime.

Tomcat / 일부 framework 의 ClassLoader leak 도 ThreadLocal 의 흔한 원인.

---

## 6. Java 의 ThreadLocal vs InheritableThreadLocal

```java
ThreadLocal<X>                    — 자식 스레드 X 못 봄
InheritableThreadLocal<X>         — fork 시 자식이 부모 값 복사
```

### 6.1 Scoped Value (21+)

```java
ScopedValue.where(USER, "alice").run(() -> {
    // USER.get() == "alice"
});
```

ThreadLocal 의 후계 — 명시적 lifetime, virtual thread 친화.

---

## 7. C 라이브러리의 옛 함수와 TLS

```c
char *strtok(s, delim);            // 내부 static state — non-reentrant
char *strtok_r(s, delim, &saveptr); // reentrant
```

`errno` 도 멀티스레드에서 TLS:
```c
extern __thread int errno;          // 실제 정의 (glibc)
```

→ 멀티스레드에서 `errno` 가 다른 스레드의 영향 없이 자기 값.

---

## 8. 성능

- TLS 접근 = nano 초 수준 (segment register)
- 보통의 mutex 보다 훨씬 빠름
- 메모리 = 변수 크기 × N 스레드

---

## 9. 라이브러리에 TLS 가 숨어있을 때

- **OpenSSL** — 옛 버전 RAND state TLS
- **glibc malloc** — arena 가 TLS-ish
- **JDBC / ORM** — connection 일부 TLS

→ 풀 환경에선 의도 확인.

---

## 10. 함정

### 10.1 풀 worker 의 TLS 누수
remove() 필수.

### 10.2 큰 TLS 변수
스레드 × 변수 크기 = 메모리 폭증.

### 10.3 InheritableThreadLocal 의 깊은 복사 누락
부모의 mutable 객체 공유 — race.

### 10.4 코루틴 / virtual thread 와 TLS
일부 라이브러리는 OS thread 기준 — 같은 코루틴이 다른 carrier 위에서 실행되면 값 다름. Scoped Value / context.

### 10.5 fork 후 TLS
process 의 fork = TLS 복사. 의도와 다를 수 있음.

### 10.6 dlopen + TLS
일부 OS / loader 에서 dynamic 로드된 라이브러리의 TLS 가 안 됨.

---

## 11. 학습 자료

- **The Linux Programming Interface** Ch. 31.4
- `man 3 pthread_setspecific`
- Java `ThreadLocal` Javadoc
- JEP 446: Scoped Values

---

## 12. 관련

- [[thread-pool]] — leak 함정
- [[threads]] — Threads hub
