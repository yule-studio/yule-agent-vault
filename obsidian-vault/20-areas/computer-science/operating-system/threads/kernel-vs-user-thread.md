---
title: "커널 스레드 vs 유저 스레드"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T09:40:00+09:00
tags:
  - operating-system
  - thread
---

# 커널 스레드 vs 유저 스레드

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 정의 + 비교 |

**[[threads|↑ Threads hub]]**

---

## 1. 정의

| | 커널 스레드 (Kernel Thread) | 유저 스레드 (User Thread) |
| --- | --- | --- |
| 관리 | 커널 (스케줄러) | 응용 / 라이브러리 |
| context | 커널이 저장 / 복원 | 응용 |
| 가시성 | OS 가 직접 본다 | OS 는 모름 (보통) |
| 비용 | 시스템 콜 필요 | 함수 호출 수준 |
| 멀티코어 | ✅ | 라이브러리 의존 |

---

## 2. 두 종류의 "커널 스레드"

용어가 혼란스러움 — 문맥 주의:

### 2.1 좁은 의미 — Linux `[kworker]`
순수 커널 공간에서만 실행. `ps aux | grep '\['` 로 보이는 `[ksoftirqd]` 등. 인터럽트 / 백그라운드 작업.

### 2.2 넓은 의미 — 사용자 task 의 커널 측 표현
`pthread_create` 로 만든 스레드도 커널 입장에선 task_struct. 보통 이 의미.

---

## 3. Linux 의 모든 스레드 = task

Linux 는 "스레드" 라는 별도 자료구조 없음. 모두 `task_struct`.
- Process = mm 독립 task
- Thread = mm 공유 task

→ 사실상 1:1 모델. 모든 사용자 스레드 = 커널이 본다.

---

## 4. 사용자 스레드 라이브러리 (옛)

### 4.1 GNU Pth, Solaris LWP, Java Green Threads (옛)

응용 안에서 스케줄. OS 는 1 스레드만 본다.

### 4.2 Coroutine / Fiber

```
부족: setjmp/longjmp, makecontext/swapcontext (POSIX)
현대: async/await, Go goroutine, Java Virtual Thread
```

---

## 5. 비교

| 측면 | 커널 스레드 | 사용자 스레드 |
| --- | --- | --- |
| 생성 비용 | 시스템 콜 (마이크로초+) | 함수 호출 (나노초) |
| 컨텍스트 스위치 | TLB / 캐시 영향 | 응용 안에서 빠름 |
| Blocking I/O | 자연스럽게 처리 | 전체 stop (N:1) 또는 처리 필요 |
| 멀티코어 | ✅ 자동 | 라이브러리 의존 |
| Preemption | 자동 (timer) | 협력 (yield) 가능 |
| 디버깅 | OS 도구 OK | 응용 도구 필요 |
| 스택 | 큰 (1-8 MB) | 작게 가능 (KB) |

---

## 6. 현대 합성 — N:M

자세히 → [[thread-models#4-nm--hybrid]]

→ 커널 스레드의 안전 + 유저 스레드의 가벼움.

---

## 7. Goroutine = 유저 스레드 + N:M

```go
go func() {
    // 가벼움 — 8KB stack, 수십만 가능
}()
```

- Go 런타임이 G 를 M (OS 스레드) 위에 스케줄
- `runtime.GOMAXPROCS(N)` — 사용할 코어 수
- Network I/O 시 자동 yield (netpoller)

---

## 8. Java Virtual Thread

```java
Thread.ofVirtual().start(() -> { ... });
```

- 카리어 스레드 (커널) 위에 mount
- I/O 시 unmount → 다른 V-thread 가 carrier 사용

---

## 9. 함정

### 9.1 "유저 스레드 = 가볍다" 단정
구현 / 사용에 따라. Java green thread 는 느렸음.

### 9.2 멀티코어 활용 X
N:1 모델은 1 코어. 현대 N:M / 1:1 권장.

### 9.3 Coroutine 안의 blocking call
다른 coroutine 도 멈춤. async 라이브러리 사용.

### 9.4 Mixed model
유저 + 커널 동시 → 디버깅 / 락 복잡.

---

## 10. 학습 자료

- **OSTEP** Ch. 27 (Threads)
- **The Linux Programming Interface** Ch. 29
- Go runtime 소스

---

## 11. 관련

- [[thread-models]]
- [[threads]] — Threads hub
