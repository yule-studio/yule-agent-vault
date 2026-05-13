---
title: "스레드 모델 — 1:1 / N:1 / N:M"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T09:35:00+09:00
tags:
  - operating-system
  - thread
  - model
---

# 스레드 모델 — 1:1 / N:1 / N:M

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 사용자 ↔ 커널 스레드 매핑 |

**[[threads|↑ Threads hub]]**

---

## 1. 정의

응용에 보이는 **사용자 스레드 (user thread)** 와 OS 가 스케줄하는 **커널 스레드 (kernel thread)** 의 매핑 방식.

---

## 2. 1:1 — Native

```
User Thread → Kernel Thread (1:1)
```

- Linux NPTL (Native POSIX Thread Library)
- Windows
- macOS / FreeBSD

### 2.1 장점
- 커널이 직접 스케줄 → blocking syscall OK
- 멀티코어 활용
- 단순

### 2.2 단점
- 생성 비용 (커널 자원)
- stack 1-8 MB × N
- context switch 시 커널 진입

→ 대부분의 현대 OS 기본.

---

## 3. N:1 — User-level (옛)

```
User N → Kernel 1
```

- 옛 Solaris LWP 이전
- 일부 임베디드 / coroutine 라이브러리

### 3.1 장점
- 매우 가벼움 (커널 비개입)
- 컨텍스트 스위치 빠름

### 3.2 단점
- **blocking syscall = 전체 stop** ⚠️
- 멀티코어 활용 X
- preemption 어려움

→ 거의 안 씀.

---

## 4. N:M — Hybrid

```
User N → Kernel M (스케줄러가 매핑)
```

- Solaris 옛
- **Go goroutine** (M:N, 현재)
- **Java Virtual Thread** (21+)
- Erlang BEAM
- 옛 NGPT (Linux 시도)

### 4.1 장점
- 가벼움 + 멀티코어
- blocking 시 다른 user 스레드 진행

### 4.2 단점
- 스케줄러 복잡 (응용 ↔ 커널)
- 디버깅 어려움
- syscall 처리 (P 분리 등 Go 의 P/M/G)

---

## 5. Go 의 G-M-P 모델 (참고)

```
G (Goroutine) — 사용자 작업
M (Machine)   — OS 스레드 (= 커널 스레드)
P (Processor) — 스케줄러 컨텍스트 (GOMAXPROCS)

각 P 가 G 큐 보유
M 이 P 와 결합해 G 실행
M 이 blocking syscall → 다른 M 가 P 인계
```

→ 수십만 goroutine + 수십 M (CPU 코어 수) 가능.

---

## 6. Java Virtual Thread (Project Loom, 21+)

```java
Thread.startVirtualThread(() -> {
    // 가벼운 스레드
});

// ExecutorService
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(...);
}
```

- 수백만 가능
- Carrier thread (커널) 위에 mount/unmount
- blocking syscall 시 carrier release

---

## 7. async/await (coroutine)

엄밀히 N:M 의 응용 레이어 변형.

- Python `asyncio`
- Rust `tokio` / `async-std`
- JavaScript Promise
- C# async/await
- Kotlin coroutine

대부분 **이벤트 루프 1 + worker N** 패턴.

---

## 8. 어떤 모델을 언제

| 언제 | 모델 |
| --- | --- |
| 일반 OS 앱 | 1:1 (pthread / java.lang.Thread / std::thread) |
| 수십만 동시 연결 (서버) | N:M (Go, Loom) 또는 async |
| 단순 CPU bound | 1:1 + 코어 수 |
| I/O 중심 | async / N:M |
| 임베디드 | RTOS task |

---

## 9. Linux NPTL — 1:1

```bash
getconf GNU_LIBPTHREAD_VERSION
# NPTL 2.39
```

- Pre-2003: LinuxThreads (옛, 결함)
- 2003+: NPTL — 진짜 POSIX 호환, futex 기반

---

## 10. Stack 크기

```bash
ulimit -s             # 기본 8 MB (스레드별 stack)

pthread_attr_setstacksize(&attr, 256 * 1024);   # 256 KB
```

거대 스레드 풀 = stack × N → 메모리 폭증.

---

## 11. 함정

### 11.1 N:1 환경에서 blocking
전체 멈춤. async / N:M.

### 11.2 1:1 에서 수십만 스레드
stack 8MB × 100K = 800 GB. 불가. 풀 / 비동기.

### 11.3 Go 의 syscall blocking
goroutine 이 syscall 차단 → M (OS 스레드) 도 차단. P 가 새 M 으로 이전 → M 폭발 가능.

### 11.4 GIL (Python) + threads
CPU bound = 효과 X. multiprocessing / asyncio.

### 11.5 Virtual thread + synchronized
Loom 의 virtual thread 가 `synchronized` 블록 안에선 pin → carrier 잡힘. `ReentrantLock` 권장.

---

## 12. 학습 자료

- **The Linux Programming Interface** Ch. 33
- **Go Scheduler** — golang.org/src/runtime
- **Java Virtual Threads** — JEP 444
- **Tokio Design**

---

## 13. 관련

- [[kernel-vs-user-thread]]
- [[threads]] — Threads hub
