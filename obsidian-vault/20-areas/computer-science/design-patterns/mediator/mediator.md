---
title: "Mediator"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, gof, behavioral, mediator, message-broker, event-bus]
---

# Mediator

**[[../design-patterns|↑ 디자인 패턴]]** · GoF Behavioral

## 1. 한 줄

**객체 간 직접 통신 대신 중개자가 조율.** N-to-N 통신을 N-to-1 + 1-to-N 으로.

## 2. 언제 쓰는가

- **다대다 통신 폭주** — N 개 component 가 서로 직결 = N² edges.
- **UI dialog 의 control 들 사이 룰** (한 버튼 활성 → 다른 입력 disable).
- **chat room / multi-player game** — 클라이언트 사이 message broker.
- **MSA event bus** — Kafka / RabbitMQ.
- **DDD application service** — domain object 사이 조율.

## 3. 구조

```
       Mediator
      /  |  |  \
     /   |  |   \
   ColleagueA  ColleagueB  ColleagueC  ColleagueD
   (서로 직접 안 알고 Mediator 에만 보고)
```

## 4. Java / Spring

```java
// Chat room mediator
interface ChatRoom {
    void send(String from, String msg);
    void register(User u);
}

class ConcreteChatRoom implements ChatRoom {
    private Map<String, User> users = new HashMap<>();
    public void register(User u) { users.put(u.name, u); }
    public void send(String from, String msg) {
        users.values().stream()
            .filter(u -> !u.name.equals(from))
            .forEach(u -> u.receive(from, msg));
    }
}

class User {
    String name;
    private ChatRoom room;

    User(String n, ChatRoom r) {
        this.name = n;
        this.room = r;
        r.register(this);
    }

    void send(String msg) { room.send(name, msg); }
    void receive(String from, String msg) { /* show */ }
}
```

**Spring `ApplicationEventPublisher`** = Mediator:
```java
@Service
class OrderService {
    @Autowired ApplicationEventPublisher publisher;

    public void place(Order o) {
        // ...
        publisher.publishEvent(new OrderPlacedEvent(o));
    }
}

@Component
class EmailListener {
    @EventListener
    void onOrderPlaced(OrderPlacedEvent e) {
        // send email
    }
}

@Component
class InventoryListener {
    @EventListener
    void onOrderPlaced(OrderPlacedEvent e) {
        // decrement
    }
}
```

- OrderService 가 다른 service 의 존재를 모름. Publisher (mediator) 가 조율.

**Spring Cloud Stream** / **Kafka** = 분산 mediator.

## 5. Python / Django / FastAPI

```python
from typing import Dict, List, Callable

class EventBus:
    def __init__(self):
        self._subscribers: Dict[str, List[Callable]] = {}

    def subscribe(self, event_name: str, handler: Callable):
        self._subscribers.setdefault(event_name, []).append(handler)

    def publish(self, event_name: str, **payload):
        for handler in self._subscribers.get(event_name, []):
            handler(**payload)

# 사용
bus = EventBus()

def send_email(order_id, **kw): print(f"email for {order_id}")
def update_inventory(order_id, **kw): print(f"decrement {order_id}")

bus.subscribe("order_placed", send_email)
bus.subscribe("order_placed", update_inventory)

bus.publish("order_placed", order_id=123)
```

**Django signals** = Mediator:
```python
from django.db.models.signals import post_save
from django.dispatch import receiver

@receiver(post_save, sender=Order)
def on_order_save(sender, instance, created, **kwargs):
    if created:
        send_email(instance)
        update_inventory(instance)
```

- Django Model 의 save 가 signal 발행, 여러 receiver 가 반응.

**Celery + Redis broker** = 분산 mediator.

## 6. TypeScript / NestJS / React

```ts
class EventBus {
  private subscribers = new Map<string, Array<(payload: any) => void>>();

  subscribe(event: string, handler: (payload: any) => void) {
    if (!this.subscribers.has(event)) this.subscribers.set(event, []);
    this.subscribers.get(event)!.push(handler);
  }

  publish(event: string, payload: any) {
    this.subscribers.get(event)?.forEach(h => h(payload));
  }
}
```

**NestJS EventEmitter**:
```ts
import { OnEvent } from '@nestjs/event-emitter';

@Injectable()
export class OrderService {
  constructor(private eventEmitter: EventEmitter2) {}

  place(order: Order) {
    this.eventEmitter.emit('order.placed', order);
  }
}

@Injectable()
export class EmailService {
  @OnEvent('order.placed')
  async sendOrderEmail(order: Order) {
    // ...
  }
}
```

**NestJS CQRS / Microservices** = full mediator + message broker (RabbitMQ / Redis / Kafka).

**React 의 Context** = 일종의 component 간 mediator:
```tsx
<UserContext.Provider value={user}>
  <Header />
  <Body />
  <Footer />
</UserContext.Provider>
```

- Header / Body / Footer 가 서로 직접 통신 없이 context 통해.

**Redux store** = mediator (action dispatch + reducer reaction).

## 7. Go

```go
type EventBus struct {
    mu          sync.RWMutex
    subscribers map[string][]func(interface{})
}

func NewEventBus() *EventBus {
    return &EventBus{subscribers: make(map[string][]func(interface{}))}
}

func (b *EventBus) Subscribe(event string, handler func(interface{})) {
    b.mu.Lock()
    defer b.mu.Unlock()
    b.subscribers[event] = append(b.subscribers[event], handler)
}

func (b *EventBus) Publish(event string, payload interface{}) {
    b.mu.RLock()
    handlers := b.subscribers[event]
    b.mu.RUnlock()
    for _, h := range handlers {
        h(payload)
    }
}
```

**NATS / Kafka / RabbitMQ** + Go client = 분산 mediator.

**Go channel + select** = 1:N / N:1 mediator 의 일부:
```go
broker := make(chan Message)

go subscriber(broker)
go subscriber(broker)

broker <- Message{...}
```

## 8. Mediator vs Observer vs Facade

| | Mediator | Observer | Facade |
| --- | --- | --- | --- |
| 통신 | N:N → N:1 + 1:N | 1:N | client 1 → subsystem |
| 결합도 | colleague 가 mediator 만 | observer 가 subject 알 수 있음 | facade 가 subsystem 단순화 |
| 양방향 | ✓ | 보통 단방향 | 단방향 |

## 9. 함정

1. **Mediator 가 god object** — 모든 통신 로직이 mediator 에. SRP 위반. domain 별 mediator 분리.
2. **circular event** — A → B event → A reaction → B again. dedup / cancel.
3. **이벤트 누락** — subscribe 전 publish. async / event log / outbox pattern.
4. **순서 보장 X** — pub-sub 가 보통 순서 보장 안 함.
5. **Django signal 의 silent failure** — receiver 의 exception 이 묻힘. logging 명시.
6. **테스트 어려움** — event-driven 의 비결정성. event 직접 assert 가 더 명확.

## 10. 관련

- [[../observer/observer]] — 1:N 단방향.
- [[../facade/facade]] — subsystem 단순화.
- [[../command/command]] — event 객체화.
