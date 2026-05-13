---
title: "Observer"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, gof, behavioral, observer, pub-sub, reactive]
---

# Observer

**[[../design-patterns|↑ 디자인 패턴]]** · GoF Behavioral

## 1. 한 줄

**1 객체의 상태 변경을 N 개 객체에 자동 통지.** Pub-Sub / Event-Driven / Reactive 의 기반.

## 2. 언제 쓰는가

- **UI 의 model-view 동기화** — model 변경 → view 자동 update.
- **Pub-Sub messaging** — Kafka / RabbitMQ / Redis Pub-Sub.
- **이벤트 driven** — 한 action 이 여러 reaction 유발.
- **Reactive programming** — RxJS / Reactor / Flow.
- **DB trigger / Django signal** — DB / ORM 의 change hook.

## 3. 구조

```
Subject                 Observer (interface)
+ subscribe(o)            + update(event)
+ unsubscribe(o)
+ notify(event)
+ state                    ▲
                           │
                  ConcreteObserverA  ConcreteObserverB
```

## 4. Push vs Pull

- **Push**: subject 가 변경 데이터를 observer 에게 전달.
- **Pull**: subject 가 "변경 있음" 만 알리고, observer 가 필요한 데이터 fetch.

## 5. Java / Spring

```java
import java.util.*;

interface Observer<T> {
    void update(T event);
}

class Subject<T> {
    private final List<Observer<T>> observers = new ArrayList<>();

    public void subscribe(Observer<T> o) { observers.add(o); }
    public void unsubscribe(Observer<T> o) { observers.remove(o); }
    public void notify(T event) {
        for (Observer<T> o : new ArrayList<>(observers)) {  // safe copy
            o.update(event);
        }
    }
}

// 사용
Subject<String> news = new Subject<>();
news.subscribe(event -> System.out.println("Email: " + event));
news.subscribe(event -> System.out.println("SMS: " + event));
news.notify("Stock alert");
```

**Spring `ApplicationEvent`** = built-in Observer:
```java
@Component
class OrderPublisher {
    @Autowired ApplicationEventPublisher publisher;
    void placeOrder(Order o) {
        publisher.publishEvent(new OrderPlacedEvent(o));
    }
}

@Component
class EmailObserver {
    @EventListener
    void onOrder(OrderPlacedEvent e) { /* send email */ }
}

@Component
class InventoryObserver {
    @EventListener
    void onOrder(OrderPlacedEvent e) { /* decrement */ }
}
```

**Spring `@Async` Event** = 비동기 observer:
```java
@EventListener
@Async
void asyncHandler(OrderPlacedEvent e) { /* ... */ }
```

**Project Reactor / RxJava** = Observer 의 reactive 확장:
```java
Flux.range(1, 10)
    .map(x -> x * 2)
    .subscribe(System.out::println);
```

## 6. Python / Django / FastAPI

```python
class Subject:
    def __init__(self):
        self._observers = []

    def subscribe(self, callback): self._observers.append(callback)
    def unsubscribe(self, callback): self._observers.remove(callback)
    def notify(self, event):
        for cb in list(self._observers):
            cb(event)

# 사용
news = Subject()
news.subscribe(lambda e: print(f"Email: {e}"))
news.subscribe(lambda e: print(f"SMS: {e}"))
news.notify("Stock alert")
```

**Django Signals** = built-in Observer:
```python
from django.db.models.signals import post_save
from django.dispatch import receiver

@receiver(post_save, sender=Order)
def on_order_save(sender, instance, created, **kwargs):
    if created:
        send_email(instance)

# 또는 custom signal
from django.dispatch import Signal
order_placed = Signal()

@receiver(order_placed)
def handle(sender, **kwargs): ...

order_placed.send(sender=OrderService, order=o)
```

**FastAPI + asyncio**:
```python
import asyncio

class AsyncSubject:
    def __init__(self):
        self.subscribers = []
    def subscribe(self, async_handler):
        self.subscribers.append(async_handler)
    async def notify(self, event):
        await asyncio.gather(*(h(event) for h in self.subscribers))
```

**Celery + Redis** = 분산 observer.

## 7. TypeScript / Node / NestJS / React

```ts
type Listener<T> = (event: T) => void;

class Subject<T> {
  private listeners = new Set<Listener<T>>();

  subscribe(l: Listener<T>): () => void {
    this.listeners.add(l);
    return () => this.listeners.delete(l);   // unsubscribe handle
  }

  notify(event: T) {
    for (const l of [...this.listeners]) l(event);
  }
}

// 사용
const news = new Subject<string>();
const unsub = news.subscribe(e => console.log('Email:', e));
news.notify('Stock alert');
unsub();
```

**Node EventEmitter**:
```ts
import { EventEmitter } from 'events';

const emitter = new EventEmitter();
emitter.on('order:placed', (order) => sendEmail(order));
emitter.on('order:placed', (order) => updateInventory(order));
emitter.emit('order:placed', order);
```

**NestJS EventEmitter / Microservices**:
```ts
@OnEvent('order.placed')
async handleOrderPlaced(payload: Order) { ... }

eventEmitter.emit('order.placed', order);
```

**React 의 useEffect + state** = Observer 의 pull 변형:
```tsx
const [user, setUser] = useState(null);

useEffect(() => {
  const unsub = authStore.subscribe((u) => setUser(u));
  return unsub;
}, []);
```

**RxJS Observable** = 가장 깊은 Observer 구현:
```ts
import { Subject, fromEvent } from 'rxjs';
import { debounceTime, map } from 'rxjs/operators';

const input$ = fromEvent(inputEl, 'input').pipe(
  debounceTime(300),
  map(e => (e.target as HTMLInputElement).value),
);

input$.subscribe(v => console.log(v));
```

## 8. Go

```go
type Observer interface {
    Update(event interface{})
}

type Subject struct {
    mu        sync.RWMutex
    observers []Observer
}

func (s *Subject) Subscribe(o Observer) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.observers = append(s.observers, o)
}

func (s *Subject) Notify(event interface{}) {
    s.mu.RLock()
    obs := append([]Observer{}, s.observers...)  // copy
    s.mu.RUnlock()
    for _, o := range obs {
        o.Update(event)
    }
}
```

**Go channel 기반 Observer** = idiomatic:
```go
type Bus struct {
    subscribers []chan Event
    mu          sync.RWMutex
}

func (b *Bus) Subscribe() <-chan Event {
    ch := make(chan Event, 16)
    b.mu.Lock()
    b.subscribers = append(b.subscribers, ch)
    b.mu.Unlock()
    return ch
}

func (b *Bus) Publish(e Event) {
    b.mu.RLock()
    defer b.mu.RUnlock()
    for _, ch := range b.subscribers {
        select {
        case ch <- e:
        default:                          // 못 받으면 drop
        }
    }
}
```

**NATS / Kafka client** = Production observer.

## 9. Anti-patterns / 함정

1. **Observer 가 long-running 동작** — notify 가 block 되면 다른 observer 도 늦어짐. async / queue.
2. **순서 의존** — observer 등록 순서로 호출 순서 가정. → priority 명시 또는 의존성 X.
3. **메모리 누수** — observer 가 unsubscribe 안 됨. WeakReference 또는 명시적 cleanup.
4. **무한 loop** — A notify → B 가 또 notify → A reaction... cycle 감지.
5. **이벤트 누락** — subscribe 전 publish. event log / outbox pattern.
6. **테스트 어려움** — async event 의 timing. test-specific synchronous bus.
7. **Django signal 의 silent error** — receiver exception 묻힘. logging 명시.

## 10. Push vs Pull trade-off

- **Push**: 빠른 반응, 그러나 observer 가 원치 않는 데이터까지 받음.
- **Pull**: observer 가 필요할 때 fetch. lazy. 그러나 staleness.

## 11. 관련

- [[../mediator/mediator]] — N:N 통신 중개.
- [[../command/command]] — event 의 reified 형태.
- [[../chain-of-responsibility/chain-of-responsibility]] — chain 의 다른 시각.
- [[../iterator/iterator]] — pull 기반 (Observer 는 push).
