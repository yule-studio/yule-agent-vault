---
title: "디자인 패턴 (Design Patterns)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T12:30:00+09:00
tags:
  - design-patterns
  - gof
  - architecture-patterns
---

# 디자인 패턴 (Design Patterns)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 수준 |

**[[../computer-science|↑ computer-science]]**

---

## 1. 한 줄 정의

**반복되는 설계 문제에 대한 검증된 해결책의 카탈로그**. 코드가 아닌 **언어**.
1994 년 GoF (Gang of Four) 책으로 정착, 이후 아키텍처 / 분산 / DDD 로 확장.

---

## 2. 역사

| 연도 | 사건 |
| --- | --- |
| 1977 | Christopher Alexander — 건축 패턴 (A Pattern Language) |
| 1987 | Kent Beck / Ward Cunningham — 패턴 운동을 SW 에 |
| 1994 | **GoF — Design Patterns** (Gamma/Helm/Johnson/Vlissides) |
| 2002 | Fowler — Patterns of Enterprise Application Architecture (PoEAA) |
| 2003 | Evans — DDD |
| 2018+ | Joshi/Fowler — Patterns of Distributed Systems |

---

## 3. GoF 23 패턴 분류

### 생성 (Creational, 5)

| 패턴 | 의도 |
| --- | --- |
| **Singleton** | 한 인스턴스 |
| **Factory Method** | 자식이 만들 타입 결정 |
| **Abstract Factory** | 관련 객체 가족 |
| **Builder** | 복잡한 객체 단계별 |
| **Prototype** | 복제 |

### 구조 (Structural, 7)

| 패턴 | 의도 |
| --- | --- |
| **Adapter** | 인터페이스 호환 |
| **Bridge** | 추상과 구현 분리 |
| **Composite** | 부분-전체 트리 |
| **Decorator** | 동적 책임 추가 |
| **Facade** | 단순 인터페이스 |
| **Flyweight** | 공유로 메모리 절약 |
| **Proxy** | 대리자 (lazy / remote / protection) |

### 행위 (Behavioral, 11)

| 패턴 | 의도 |
| --- | --- |
| **Chain of Responsibility** | 처리자 체인 |
| **Command** | 요청을 객체로 |
| **Iterator** | 순회 추상화 |
| **Mediator** | 상호작용 중개 |
| **Memento** | 상태 저장 / 복원 |
| **Observer** | 1-N 통지 |
| **State** | 상태 별 행동 |
| **Strategy** | 알고리즘 교체 |
| **Template Method** | 알고리즘 골격 |
| **Visitor** | 구조 + 연산 분리 |
| **Interpreter** | 문법 평가 |

---

## 4. 자주 쓰는 GoF 패턴

### 4.1 Singleton

```python
class Logger:
    _instance = None
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
```

**주의** — 멀티스레드, 테스트 어려움. 의존성 주입이 보통 더 낫다.

### 4.2 Factory Method

```python
from abc import ABC, abstractmethod

class Logger(ABC):
    @abstractmethod
    def log(self, msg): ...

class FileLogger(Logger):
    def log(self, msg): print(f"[file] {msg}")

class ConsoleLogger(Logger):
    def log(self, msg): print(msg)

class LoggerFactory:
    @staticmethod
    def create(kind):
        if kind == "file": return FileLogger()
        return ConsoleLogger()
```

### 4.3 Builder

```python
class HttpRequest:
    def __init__(self):
        self.url = None
        self.method = "GET"
        self.headers = {}

class Builder:
    def __init__(self): self.req = HttpRequest()
    def url(self, u): self.req.url = u; return self
    def method(self, m): self.req.method = m; return self
    def header(self, k, v): self.req.headers[k] = v; return self
    def build(self): return self.req

req = (Builder()
       .url("https://api.com")
       .method("POST")
       .header("Auth", "Bearer ...")
       .build())
```

### 4.4 Adapter

```python
class LegacyPayment:
    def make_payment(self, amount): ...

class StripeAPI:
    def charge(self, dollars, currency): ...

class StripeAdapter(LegacyPayment):
    def __init__(self, stripe): self.stripe = stripe
    def make_payment(self, amount):
        self.stripe.charge(amount, "USD")
```

### 4.5 Decorator

```python
def log(func):
    def wrapper(*args, **kwargs):
        print(f"calling {func.__name__}")
        return func(*args, **kwargs)
    return wrapper

@log
def add(a, b): return a + b
```

### 4.6 Observer

```python
class Subject:
    def __init__(self):
        self.observers = []
    def subscribe(self, o): self.observers.append(o)
    def notify(self, event):
        for o in self.observers: o.update(event)

class EmailNotifier:
    def update(self, e): print(f"📧 {e}")

class SmsNotifier:
    def update(self, e): print(f"📱 {e}")

s = Subject()
s.subscribe(EmailNotifier())
s.subscribe(SmsNotifier())
s.notify("New order")
```

Pub-Sub, Event-Driven 의 기반.

### 4.7 Strategy

```python
class Sorter:
    def __init__(self, strategy): self.strategy = strategy
    def sort(self, data): return self.strategy(data)

quicksort = lambda d: sorted(d)
reverse_sort = lambda d: sorted(d, reverse=True)

Sorter(quicksort).sort([3,1,2])
Sorter(reverse_sort).sort([3,1,2])
```

함수 일급 객체인 언어 (Python/JS) 에서는 함수만으로 충분.

### 4.8 Command

```python
class Command:
    def execute(self): ...
    def undo(self): ...

class Move(Command):
    def __init__(self, x, y, dx, dy):
        self.x, self.y, self.dx, self.dy = x, y, dx, dy
    def execute(self): return (self.x + self.dx, self.y + self.dy)
    def undo(self): return (self.x, self.y)
```

Undo / Redo / Macro / Queue.

### 4.9 State

```python
class TrafficLight:
    def __init__(self): self.state = "RED"
    def next(self):
        self.state = {"RED": "GREEN", "GREEN": "YELLOW", "YELLOW": "RED"}[self.state]
```

복잡해지면 State 객체로 분리.

### 4.10 Template Method

```python
class Report:
    def generate(self):
        data = self.fetch()
        result = self.process(data)
        self.write(result)
    def fetch(self): ...        # 자식 구현
    def process(self, d): ...   # 자식 구현
    def write(self, r): ...     # 자식 구현
```

### 4.11 Iterator

Python 의 `__iter__` / `__next__`. for-loop 가 이걸 씀.

### 4.12 Proxy

```python
class ImageProxy:
    def __init__(self, path): self.path = path; self.image = None
    def display(self):
        if self.image is None: self.image = self._load()
        self.image.display()
    def _load(self): return Image(self.path)
```

Lazy loading, 캐싱, 권한 검증.

---

## 5. 패턴 안티유즈

### 5.1 Anti-Pattern: Singleton 남용
- 글로벌 상태 = 테스트 지옥
- 의존성 주입이 거의 항상 낫다

### 5.2 Anti-Pattern: 모든 곳에 Factory
- 단순한 생성에 팩토리 X — YAGNI

### 5.3 Anti-Pattern: 깊은 Decorator 체인
- 디버깅 / 추적 어려움

### 5.4 Anti-Pattern: Visitor 의 강제
- 함수형 패턴 매칭이 있는 언어에선 불필요

### 5.5 Anti-Pattern: 패턴을 위한 패턴
- "이걸 Strategy 로 만들어야겠다" 가 아닌 "이 변경이 자주 일어난다" 가 동기

---

## 6. 함수형 패턴 (FP)

GoF 는 OOP 기반. 함수형에선 다른 모습:

| GoF | FP 대응 |
| --- | --- |
| Strategy | High-order function |
| Command | Closure |
| Iterator | Lazy sequence (generator) |
| Observer | Stream / Reactive (Rx) |
| Visitor | Pattern matching |
| Template Method | Higher-order function |
| Decorator | Function composition |
| State | Monad / Reducer |

---

## 7. 엔터프라이즈 패턴 (PoEAA, Fowler 2002)

### 7.1 도메인 로직

- **Transaction Script** — 절차적
- **Domain Model** — 객체 + 행동
- **Table Module** — 테이블 당 한 객체
- **Service Layer** — 트랜잭션 경계

### 7.2 데이터 소스

- **Active Record** — Rails / Django
- **Data Mapper** — Hibernate / SQLAlchemy
- **Repository** — DDD
- **Unit of Work** — 변경 추적

### 7.3 객체-관계 매핑

- **Identity Map** — 메모리 캐시
- **Lazy Load**
- **Foreign Key Mapping**
- **Inheritance Mapping** — Single Table / Class Table / Concrete Table

### 7.4 웹 / 분산

- **Front Controller** — 한 진입점
- **Page Controller** — 페이지 당
- **Model-View-Controller**
- **Remote Facade**
- **Data Transfer Object (DTO)**

---

## 8. 동시성 패턴

- **Producer-Consumer** — 버퍼 + condvar
- **Reader-Writer** — RW lock
- **Thread Pool** — 미리 생성한 스레드
- **Future / Promise** — 비동기 결과
- **Actor** — 메시지 전달 (Erlang, Akka)
- **CSP** (Communicating Sequential Processes) — Go channel
- **Reactor** — 이벤트 루프 (Node.js, Nginx)
- **Proactor** — 비동기 완료 (Windows IOCP)

---

## 9. 분산 / MSA 패턴

[[../distributed-systems/distributed-systems#10 분산 패턴]] 참조.

- Service Discovery
- Circuit Breaker
- Bulkhead
- Saga (Choreography / Orchestration)
- CQRS
- Event Sourcing
- Outbox
- Sidecar
- API Gateway
- BFF (Backend for Frontend)
- Strangler Fig — 레거시 점진적 교체

---

## 10. 함정 / 안티패턴 (총괄)

### 함정 1 — 패턴 사냥
"패턴 적용" 이 목표가 됨. 문제부터.

### 함정 2 — 책에 있는 그대로
언어 / 컨텍스트에 맞게 변형 필요.

### 함정 3 — 이름만 알고 의도 모름
"이건 Strategy 패턴" 만 알고 왜 인지 모름.

### 함정 4 — 패턴 = 좋은 코드 X
나쁜 코드에 패턴 적용해도 여전히 나쁨.

### 함정 5 — 너무 많은 패턴 동시 적용
신규 코드를 읽을 수 없게 만듦.

---

## 11. 면접 질문

1. **Singleton 의 문제와 대안**.
2. **Factory vs Abstract Factory vs Builder**.
3. **Decorator vs Proxy**.
4. **Observer 의 단점**.
5. **Strategy vs State**.
6. **Template Method vs Strategy**.
7. **Adapter vs Facade**.
8. **자주 쓰는 패턴 3 가지와 이유**.
9. **함수형 언어에서의 GoF**.
10. **MSA 의 Saga 패턴**.

---

## 12. 학습 자료

- **Design Patterns** (GoF, 1994) — 원전
- **Head First Design Patterns** — 시각적
- **Refactoring to Patterns** (Joshua Kerievsky)
- **Patterns of Enterprise Application Architecture** (Fowler 2002)
- **Game Programming Patterns** (Robert Nystrom) — 무료 https://gameprogrammingpatterns.com
- [refactoring.guru](https://refactoring.guru/design-patterns) — 시각화 우수
- **Patterns of Distributed Systems** (Joshi)

---

## 13. 관련

- [[../software-engineering/software-engineering]] — SOLID
- [[../distributed-systems/distributed-systems]] — MSA 패턴
- [[../computer-science|↑ computer-science]]
